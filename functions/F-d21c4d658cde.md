<!-- Trusted provenance: F-d21c4d658cde; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d21c4d658cde

## Purpose

F-d21c4d658cde updates a position-like record using a signed requested value and a count obtained indirectly through that record. It then derives two neighboring-position fields, subject to a second dependency-controlled gate.

The established behavior uses two distinguished values:

- **−2** is the result for negative requests other than −3.
- **−3** is preserved when explicitly requested and is also selected for a positive request that is at least the observed count.

Their interpretation as “before the beginning” and “after the end” is inferred intent, not an established external contract. The observable contract is the numeric normalization and ordered field updates described below.

The operation is not an atomic replacement of the record. Its dependency can prevent all direct updates or permit the selected-position update while preventing both neighboring-position updates.

## Interface

The function accepts:

1. **A record address.** The record contains four integer-sized slots. Its first slot supplies an address used to access a separate count.
2. **A signed requested value.** This controls the selected-position update.

There is no returned value. Results are observable through record writes and any effects of dependencies.

### Record layout

The supplied representation supports the following four-byte-slot layout:

| Byte offset | Contract name | Use |
|---|---|---|
| 0 | Count-source address | Address of another object; its count is read at byte offset 4 |
| 4 | Preceding-position field | Derived output |
| 8 | Selected-position field | Normalized request, subsequently reread |
| 12 | Following-position field | Derived output |

The count-source address is represented in an integer slot and used for address arithmetic and dereferencing. It is not itself a count.

The separate object must expose a readable signed integer at byte offset 4 from the address obtained from the record. No other part of that object is directly inspected.

### Widths and representation

The behavioral tables below use the recovered **32-bit signed-integer model**: four-byte integer slots with signed comparisons. The fixed byte displacement used for the count is 4. The source does not independently establish the original language ABI, pointer width, byte order, or overflow semantics.

In particular:

- The address-bearing first slot must support the represented address on the target platform.
- The requested value, count, and three position fields are signed integer values.
- Negative sentinel values are compared as signed values, not unsigned bit patterns.
- No byte-order conversion or explicit narrowing operation is shown.

Exact behavior outside this recovered representation is **documented_only**.

### Preconditions

For every access actually reached:

- The record address must designate suitably aligned, accessible storage.
- Its address-bearing slot must be readable when the function needs the count.
- The selected-position slot must be writable if the first gate permits progress, and readable if the second gate permits derivation.
- The neighboring-position slots must be writable when their corresponding updates are reached.
- Each count-source address actually loaded must support a readable integer at byte offset 4.

There are no explicit null checks, bounds checks on the record storage, or address-validity checks. A gate returning zero can avoid accesses that would otherwise require these preconditions, but the gate’s own access requirements are unknown.

## Behavior

### 1. Initial gate

The first action is an invocation of **F-392d97f6cbd4**, with no explicit arguments.

- If its returned integer is zero, F-d21c4d658cde returns immediately.
- Otherwise, request normalization proceeds.

On the zero-result path, the supplied body performs no direct record reads or writes. This does **not** establish that the record or other state remains unchanged: F-392d97f6cbd4 may have side effects.

### 2. Request normalization

Following a nonzero first gate result, the function writes exactly one value to the selected-position field:

| Requested value | Count access during normalization | Value written |
|---|---|---|
| −3 | None | −3 |
| Any other negative value | None | −2 |
| 0 | None | 0 |
| Positive and less than the observed count | Read count through the record | Requested value |
| Positive and at least the observed count | Read count through the record | −3 |

For a positive request, the count comparison is signed. Thus any positive request becomes −3 when the observed count is zero or negative.

For the other request classes, no count is read during normalization. In particular, zero is stored as zero even if the count is zero or negative. The later derivation does not necessarily replace that selected value.

When a count access is required, the function obtains the count-source address from the record and reads the integer at that address plus 4 before writing the selected-position field.

### 3. Second gate

After the selected-position write, the function invokes **F-392d97f6cbd4** again.

- If this result is zero, the function returns without directly updating either neighboring-position field.
- If this result is nonzero, neighboring-position derivation proceeds.

There is no rollback of the selected-position write.

The second dependency invocation occurs after that write and before the selected-position reread. Consequently, the later derivation is governed by the selected-position value actually observed after the dependency returns, not necessarily the value just stored.

The implementation does not establish that either dependency invocation preserves the record, count-source object, globals, ownership state, or any other reachable memory.

### 4. Inputs to neighboring-position derivation

After a nonzero second gate result, the function reads, in this order:

1. The selected-position field.
2. The count-source address and the count at its byte offset 4.

Call these observed values the **selected value** and the **derivation count**.

The derivation count is a new read. It is not a saved copy of any count used during normalization. Dependency effects, aliasing, or external changes may therefore make the two observations differ.

### 5. Nonpositive derivation count

When the derivation count is less than 1:

| Selected value | Ordered writes |
|---|---|
| −2 | Following-position field becomes −2; preceding-position field becomes −2 |
| Any other value | Following-position field becomes −3; preceding-position field becomes −3 |

The selected-position field is not directly changed during this stage.

For example, a normalized request of zero with a zero count leaves the selected-position field at zero while assigning −3 to both neighboring-position fields, assuming no intervening dependency effects.

### 6. Positive derivation count

When the derivation count is at least 1, behavior depends on the selected value.

#### Selected value −2

The function writes:

1. Following-position field: 0.
2. Preceding-position field: −2.

It then returns.

#### Selected value −3

The function writes:

1. Following-position field: −3.
2. Preceding-position field: one less than a **newly read count**.

The count used for the second write is obtained by rereading the count-source address and the count at its byte offset 4 **after** the following-position write.

This reread is significant. The earlier positive-count decision does not guarantee that the newly read count is positive or unchanged. Nor does the supplied body establish that the following-position write cannot affect storage involved in that reread.

#### Any other selected value

The function performs these ordered updates:

1. Write the selected value plus 1 to the following-position field.
2. If that computed value is at least the derivation count, overwrite the following-position field with −3.
3. Write the selected value minus 1 to the preceding-position field.
4. If that computed preceding value is negative, overwrite the preceding-position field with −2.

The comparisons use the selected value and derivation count already read for this stage. There is no additional count read in this case.

The intermediate writes are part of the established access sequence. For example, at the final in-range position, the following-position field is first assigned the count and then overwritten with −3.

### Arithmetic limits

For ordinary normalized inputs and stable, nonaliased storage, the positive-count paths keep the relevant arithmetic within the signed 32-bit range:

- An accepted positive request is strictly below the count.
- A selected value of zero yields 1 and −1 during derivation.
- A stable positive count can safely be reduced by 1 in the −3 case.

Those conditions are not enforced across dependency calls or rereads.

The body provides no explicit overflow handling for:

- Adding 1 to the maximum signed selected value.
- Subtracting 1 from the minimum signed selected value.
- Subtracting 1 from a reread count equal to the minimum signed value.

The exact result in those circumstances is **documented_only**. Wrapping, saturation, trapping, or another language/runtime consequence must not be assumed.

## Side effects

### Direct memory effects

The function may directly write:

- The selected-position field, once after a successful initial gate.
- The following-position field, once or twice after a successful second gate.
- The preceding-position field, once or twice after a successful second gate.

It does not directly assign the count-source address or the count. Aliased storage can nevertheless cause a position-field write to affect later address or count observations.

Direct neighboring-position writes always occur in following-then-preceding order. There is no combined structure write, transactional commit, or rollback.

### Dependency effects

F-392d97f6cbd4 is invoked:

- Once if the first result is zero.
- Twice if the first result is nonzero.

The second invocation is reached after the selected-position assignment, regardless of which normalization case was taken.

Any dependency-visible state changes remain part of the operation’s effects. No preservation guarantee can be inferred from the absence of explicit arguments.

### Ownership and failures

No direct allocation, release, reference-count operation, ownership transfer, I/O, or explicit error reporting appears in the supplied body.

This does not establish corresponding guarantees for dependencies. The function supplies no status indicating whether either gate blocked progress. Invalid memory and unsupported arithmetic have no explicit recovery path in the body.

## Dependencies

### F-392d97f6cbd4

This dependency supplies the two gates.

Established facts:

- It is invoked without explicit arguments.
- Its integer result is tested for equality to zero.
- Zero causes an immediate return at the relevant stage.
- Every nonzero result permits progress; no other result distinction is made.

Unknowns include its purpose, side effects, memory accesses, ownership behavior, and whether repeated invocations normally return consistent results. Calling it a validity, readiness, or synchronization check would be inferred intent only.

### F-e409dfa213c9

The supplied representation associates this identifier with the continuation that starts at the second gate and performs neighboring-position derivation.

It shows an intra-body transfer to that continuation, not a distinct ordinary call with an independently demonstrated calling convention. No additional arguments, return value, preservation guarantees, or independently callable interface are established.

The continuation’s observable behavior is specified above. Treating it as an unrelated extra invocation would add behavior not established by the supplied representation.

## Acceptance examples

These examples are conditional specifications, not observed executions. Unless stated otherwise, assume:

- Both gates return nonzero.
- Dependencies do not modify relevant storage.
- Record and count-source storage do not alias.
- The count remains stable.
- All required accesses are valid and arithmetic is representable.

Final values are listed as **selected, preceding, following**.

| Count | Request | Expected final values |
|---:|---:|---|
| 5 | −3 | −3, 4, −3 |
| 5 | −2 | −2, −2, 0 |
| 5 | −1 | −2, −2, 0 |
| 5 | −10 | −2, −2, 0 |
| 5 | 0 | 0, −2, 1 |
| 5 | 2 | 2, 1, 3 |
| 5 | 4 | 4, 3, −3 |
| 5 | 5 | −3, 4, −3 |
| 5 | 6 | −3, 4, −3 |
| 1 | 0 | 0, −2, −3 |
| 0 | −2 | −2, −2, −2 |
| 0 | 0 | 0, −3, −3 |
| 0 | 1 | −3, −3, −3 |
| −4 | −3 | −3, −3, −3 |

Additional conditional acceptance cases:

- **First gate returns zero:** no direct record access occurs, and no second gate invocation occurs. Any dependency effects remain possible.
- **First gate succeeds, second gate returns zero:** normalization is performed, but neither neighboring-position field is directly written.
- **Count changes between gates:** with request 4, an initial count of 5, and a derivation count of 3, the selected value remains 4. The following field is written as 5 and then −3; the preceding field becomes 3.
- **Second gate changes the selected field:** with a positive derivation count, if that dependency changes the selected field to −2, the derived fields become preceding −2 and following 0.
- **Count changes during the −3 path:** if the derivation count is 5 but the post-following-write reread is 8, the preceding field becomes 7, not 4.
- **Second gate introduces selected value −1:** with count 5, the general derivation case produces following 0 and preceding −2. This value is not produced by ordinary request normalization, but the continuation does not reject it.

## Uncertainties

The following remain **documented_only**:

- Original ABI, independently verified integer and pointer widths, and byte order.
- Domain meaning of the record, count, and sentinel values.
- The semantic purpose and full effects of F-392d97f6cbd4.
- Whether record/count aliasing or concurrent mutation is permitted by callers.
- Whether accesses are atomic, volatile, synchronized, or externally observable individually.
- Exact overflow and invalid-address behavior.
- Independent entry requirements for F-e409dfa213c9.
- Whether callers impose stronger invariants that exclude unusual selected values, negative counts, or state changes between reads.

The apparent position-and-neighbor interpretation is useful explanatory intent, not a substitute for the numeric contract.

## Evidence

Established static evidence consists of the supplied function representation and opaque dependency mapping. It supports the gate ordering, signed comparisons, sentinel normalization, address-plus-4 count reads, record-field writes, reread behavior, and shared continuation described here.

The field names are explanatory contract names, not recovered domain terminology. The four-byte layout is the recovered model; broader platform guarantees are not independently established.

No runtime tests have been performed. Acceptance examples are derived conditionally from static control flow and arithmetic, not execution results.
