<!-- Trusted provenance: F-2fb582eca75a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2fb582eca75a

## Purpose

F-2fb582eca75a conditionally invokes a final operation after two search-like calls, two measurements, and two position adjustments.

The first search operand comes from an indexed table. The second comes from a field in shared state. Each search must return a value other than the distinguished 32-bit value −1 for processing to continue.

The normal return is:

- **0** if either search returns −1.
- **1** after the final operation returns normally.

A return of 1 establishes that the final operation was invoked and returned normally. Its result is ignored, so 1 does not independently establish success of that operation.

The apparent intent is to locate successive markers, advance and adjust a position, locate a terminal boundary, and perform an operation using those values. Marker, position, and boundary meanings are inferred from data flow; dependency implementations are unavailable.

The temporary position is exposed by address to an adjustment dependency. Later dependencies may therefore be able to modify it through an alias. In particular, the second position addition need not use the second search result, and the final operation need not receive the position previously supplied to the terminal lookup.

## Interface

### Arguments

| Argument | Visible representation | Established role |
|---|---|---|
| Context | Opaque 32-bit value | Passed to both adjustment calls. |
| Selector | Signed 32-bit integer | Selects an entry using a four-byte stride. |
| Subject | Opaque 32-bit value | Passed to both searches, both adjustments, the terminal lookup, and the final operation. |
| Final option | Signed 32-bit integer | Passed unchanged to the final operation. |

The concrete types of context and subject are unknown. They may encode addresses, handles, or other values. This function does not directly dereference the subject.

The final option is initially copied into temporary 32-bit position storage. The first search result subsequently overwrites that storage. There is no visible use of the initial copy as a search starting position.

### Result and writable storage

The function returns a 32-bit 0 or 1 on the shown normal paths.

It passes the address of the same temporary 32-bit position storage to F-6556bc069ccf on each adjustment call. Writes through that address affect later processing. The source does not establish that the address remains private to that dependency or that modifications occur only during the adjustment calls.

### Preconditions for normal execution

Reached memory accesses must be valid:

- F-58e65610fe16 must provide a usable table base.
- The selected four-byte entry must be readable on each reached table access.
- The shared-state base accesses and the field reads at byte offset 0x48 must identify readable storage.
- The captured shared object must remain readable when its field is accessed after the second search.

Arguments must also satisfy the unknown requirements of the invoked dependencies.

There are no visible null checks, selector bounds checks, checked arithmetic operations, or exception handlers. Invalid memory, arithmetic overflow, and dependency failures other than the explicit sentinel results have no established recoverable behavior here.

## Behavior

### 1. Obtain the table and perform the first search

F-58e65610fe16 is called without arguments. Its result supplies the table base.

The function selects an entry at the table base plus four times the selector and reads a 32-bit operand. It calls F-0d3d19f29b0f with:

1. The subject.
2. The operand just read.
3. Zero.

The returned value is stored in the temporary position.

If the stored value is −1, the function returns 0 immediately. No measurement, adjustment, second search, terminal lookup, fallback, or final operation is reached.

Only −1 is rejected. Other negative results proceed.

### 2. Measure the first operand and adjust the position

After a non-sentinel first result, the function rereads the selected table entry using the previously obtained table base and the same selector. It passes the reread operand to F-1cb585febd84.

This second entry read is not replaced by a cached copy of the first operand:

- If the first search changes that entry, the measurement receives the changed value.
- A change that would make the table provider return a different base does not redirect this reread. The provider is not called again.

After F-1cb585febd84 returns, its result is added to the current temporary position, and the sum replaces that position. On the visible data flow, the temporary still contains the first search result: its address has not yet been supplied to a dependency by this invocation.

F-6556bc069ccf is then called with the context, subject, and address of the temporary position. Its return value is ignored.

The function performs no validation after the adjustment. A written value of −1, another negative value, or an out-of-range value is not independently rejected. The position value evaluated for the next search includes changes made through the supplied address.

This call exposes an alias to the temporary. No available dependency contract excludes retention or sharing of that alias.

### 3. Obtain the second operand and perform the second search

After the first adjustment returns, the function captures the current shared-state base for later use.

The second search operand is obtained through a separate access to the shared-state base and a 32-bit read at byte offset 0x48. The source does not express this operand read as a reuse of the captured base. No intervening dependency call is shown between the capture and operand read; concurrency and external mutation rules are unknown.

F-a21141daa905 receives:

1. The subject.
2. The shared-field operand just read.
3. The temporary position evaluated for this call.

Its return value replaces the temporary position. Consequently, a modification to that temporary during the search call is superseded by the returned result when the assignment occurs.

If the stored result is −1, the function returns 0. Earlier calls and their effects are not undone. The second measurement, second adjustment, terminal lookup, fallback, and final operation are not reached.

### 4. Measure the second operand and adjust the current position

After a non-sentinel second result, the function reads the 32-bit field at offset 0x48 from the shared-state base captured before the second search. It passes this freshly read field value to F-1cb585febd84.

The operand can differ from the one supplied to the second search:

- Changes to the field in the captured object are visible to this later read.
- Replacement of the global shared-state base during the second search does not redirect this read to the replacement object.
- If the captured object is released or otherwise becomes unreadable, normal behavior is not established.

After the measurement returns, its result is added to the **then-current temporary position**.

The addend is not unconditionally the second search result. The earlier adjustment received the temporary’s address; F-1cb585febd84 could modify that storage through an alias retained or shared by F-6556bc069ccf. Such a modification during the second measurement is consumed by the subsequent addition.

If no such modification occurs between assignment of the second search result and the addition, the addition uses that search result. Otherwise, it uses the modified value. There is no intervening sentinel check.

F-6556bc069ccf is then called again with the context, subject, and address of the same temporary storage. Its return value is ignored. Its writes again affect subsequent processing without validation.

### 5. Resolve the terminal value

F-a4621b603aad is called with:

1. The subject.
2. The integer constant 0x26, decimal 38.
3. The temporary position evaluated before this call.

The constant corresponds to an ampersand in ASCII. Its interpretation by the dependency is not established.

If the lookup returns anything other than −1, that result becomes the terminal value.

If it returns −1, F-d8cbf9aa79a9 is called without arguments. Its result replaces the terminal value without validation. A fallback result of −1 or another negative value is not rejected.

The terminal value and the temporary position are separate values. Neither the lookup nor the fallback is shown receiving the temporary’s address directly, but either could access an alias exposed by an earlier adjustment. The position after these calls is therefore not guaranteed to equal the position supplied to the lookup.

### 6. Invoke the final operation

After the lookup and any fallback return, F-6ec174765772 receives:

1. The subject.
2. The temporary position as read for this final call.
3. The resolved terminal value.
4. The original final option.

The second argument reflects any modifications to the temporary made during the lookup or fallback. It is not necessarily the position left by the second adjustment.

The final operation’s return value is ignored. If it returns normally, F-2fb582eca75a returns 1.

No visible check requires the terminal value to be at least the position, either value to be nonnegative, or the pair to describe a valid region of the subject.

### Widths and arithmetic

The visible representation uses 32-bit argument values, table entries, shared-field values, temporary position storage, and normal return values.

The selector is signed, with a four-byte entry stride. The shared-field offset is 0x48 bytes, or 72 bytes.

There are two position additions:

- The first measurement result plus the temporary position read after that measurement returns. The visible flow has not yet exposed this temporary by address.
- The second measurement result plus the temporary position read after that measurement returns. This position may have changed through the previously exposed alias and need not equal the second search result.

Neither addition has visible overflow detection, saturation, or a wider intermediate. The selector scaling and address calculation are likewise unchecked. The supplied representation does not establish a portable policy for overflowing signed arithmetic; wrapping or another outcome must not be asserted as established behavior.

## Side effects

The function directly writes its temporary position and directly reads table and shared-state storage. Its ordered direct nonlocal reads are:

1. The selected table entry before the first search.
2. The selected table entry after the first search succeeds.
3. The shared-state base captured after the first adjustment.
4. The separate shared-base access and field read for the second search.
5. The field in the captured shared object after the second search succeeds.

Dependencies can introduce additional effects between these reads.

The temporary’s address is supplied to F-6556bc069ccf twice on the successful path. Modifications through that address can affect later behavior beyond the immediate adjustment calls:

- The second search’s assigned return overwrites any temporary modification made during that search.
- A modification during the second measurement can change the operand of the second addition.
- A modification during the terminal lookup or fallback can change the final operation’s second argument without changing the position already supplied to the lookup.

The function itself shows no allocation, release, or explicit ownership transfer. This does not establish that dependencies preserve storage, retain no references, or perform no allocation or release.

The temporary belongs to this invocation. Retention of its address is not excluded by the supplied evidence; safe dereferencing after its lifetime ends is not established.

Returning 0 after the second search does not roll back earlier dependency effects. Abnormal termination may also leave partial effects; no cleanup or recovery mechanism is visible.

## Dependencies

| Opaque function ID | Established use |
|---|---|
| F-58e65610fe16 | Called once without arguments to obtain the indexed table base. |
| F-0d3d19f29b0f | Receives subject, selected table operand, and zero. Its returned −1 causes return 0. |
| F-1cb585febd84 | Receives a freshly read operand after each successful search. Its return is added to the temporary position read after the call. On the second invocation, that position may have changed through an exposed alias. |
| F-6556bc069ccf | Receives context, subject, and a writable pointer to the temporary position. Called twice on the successful path; return values are ignored. |
| F-a21141daa905 | Receives subject, shared-field operand, and the position evaluated before the call. Its result replaces the temporary; −1 causes return 0. |
| F-a4621b603aad | Receives subject, 0x26, and the position evaluated before lookup. Its returned −1 selects the fallback. It is not established to preserve the aliased temporary. |
| F-d8cbf9aa79a9 | No-argument fallback called only after lookup returns −1. Its result is used unchecked. It is not established to preserve the aliased temporary. |
| F-6ec174765772 | Receives subject, the position read after lookup and fallback, the resolved terminal value, and the original final option. Its result is ignored. |

“Search,” “measurement,” “adjustment,” and “terminal lookup” are descriptive roles inferred from use. They do not establish encodings, units, ownership, range inclusivity, or dependency success semantics.

## Acceptance examples

These are conditional acceptance criteria, not executed tests. They assume valid memory, normal dependency returns, and representable arithmetic.

### First search failure

If F-0d3d19f29b0f returns −1:

- The table provider and first search have been called.
- The selected entry has been read once.
- The function returns 0.
- No later dependency is called.

### Second search failure

Suppose the first search returns 8, the first measurement returns 3, and the first adjustment changes the resulting position from 11 to 15.

The second search receives 15 as its third argument. If it returns −1, the function returns 0 without a second measurement, second adjustment, terminal lookup, fallback, or final operation. Earlier effects remain.

### Successful path with unchanged lookup position

Suppose:

- The first search returns 8 and its measurement returns 3.
- The first adjustment leaves the position at 11.
- The second search returns 20.
- The second measurement returns 2 and does not modify the temporary through an alias.
- The second adjustment changes 22 to 24.
- The terminal lookup returns 40 and does not modify the temporary.

The lookup receives subject, 0x26, and 24. The final operation receives subject, 24, 40, and the original final option. No fallback is called. Normal return from the final operation is followed by return 1.

### Aliased mutation during the second measurement

Suppose the second search returns 20. During the second measurement, a previously exposed alias is used to change the temporary to 30, and the measurement returns 2.

The position supplied by address to the second adjustment initially contains 32, not 22. The function does not restore the second search result before addition.

### Lookup changes the eventual final position

Suppose the second adjustment leaves the temporary at 24. The lookup receives 24, changes the temporary through an alias to 29, and returns 40.

The final operation receives 29 as its second argument and 40 as its third. The earlier lookup still received 24.

### Fallback changes the eventual final position

Suppose the lookup receives 24, changes the temporary to 25, and returns −1. The fallback then changes the temporary to 31 and returns −1.

The final operation receives 31 as its second argument and −1 as its third. Neither value is rejected here. If the final operation returns normally, the function returns 1.

### Operand rereads and negative results

If the first search replaces the selected table entry, the first measurement receives the replacement from the original table base.

If the second search replaces the global shared base, the second measurement still reads the captured object, provided it remains valid.

A search result of −2 does not take the explicit failure branch. Subsequent processing depends on the dependency behavior and arithmetic; this function supplies no additional rejection.

## Uncertainties

The following remain **documented_only**:

- Concrete types, ownership, and lifetime requirements for context, subject, operands, and shared storage.
- Table size, valid selector range, and validity of negative selectors.
- Position and measurement units, including whether values are offsets, addresses, or another representation.
- Terminal-value meaning and range inclusivity.
- Fallback meaning and its relationship to the subject despite having no explicit subject argument.
- Exact overflow and address-arithmetic behavior.
- Whether dependencies retain or share the temporary’s address, and which later calls can access it.
- Dependency effects on globals, table entries, shared fields, memory lifetime, and ownership.
- Reentrancy, concurrency, and synchronization.
- Failure reporting by the final operation.
- Abnormal termination behavior beyond the explicit sentinel branches.

No dependency is assumed to preserve globals, preserve aliased temporary storage, or own or release memory.

## Evidence

Established behavior is limited to the supplied static function body: argument widths and forwarding, four-byte indexed reads, shared-field offset, ordered calls, post-call position additions, address exposure, sentinel comparisons, fallback selection, final argument evaluation, and return constants.

The alias-sensitive behavior follows from exposing temporary storage before later calls and reading that storage after those calls return. Actual retention or mutation by a dependency is not established; absence of such mutation is likewise not established.

The marker-and-range interpretation is inferred intent only. No dependency implementations or runtime observations were supplied. No runtime tests have been performed.
