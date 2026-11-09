<!-- Trusted provenance: F-bada1d21b9de; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bada1d21b9de

## Purpose

F-bada1d21b9de conditionally applies a three-word value obtained from an associated object, then selects one of two update paths:

- A broader notification path when a post-update state field is nonzero or either of two captured values differs from the requested values.
- A narrower region-update path when those conditions are absent but a third captured value differs.

This is the established control-flow behavior. Interpreting the three words as position, size, orientation, or another application-specific property is not established. Likewise, the apparent region operations suggest invalidation or redraw intent, but their exact semantics depend on undocumented callees.

The return value is path-dependent. It is not consistently a success flag or consistently the input object.

## Interface

### Inputs

The visible interface has three arguments:

1. **Object address.** A pointer to a mutable object with 32-bit fields and an indirect dispatch table.
2. **Opaque 32-bit argument.** No explicit use is visible in the supplied representation.
3. **32-bit input value.** This value does not determine any visible branch: when the associated-object field is nonzero, it is replaced by an object field before use; when the associated-object field is zero, the guarded operation is skipped.

The second and third arguments should not be assigned additional semantics without further evidence.

### Object layout used by this function

Offsets below are byte offsets from the first argument.

| Offset | Visible role |
|---|---|
| 0x00 | Address of the dispatch table used for four indirect calls |
| 0x34 | Optional related-object address |
| 0x38 | Associated-object address used for lookup |
| 0x44 | Enable value used when the associated-object address is nonzero |
| 0x80 | Address of data used in constructing a temporary; later also read as a 32-bit transformation argument |
| 0x84 | 32-bit transformation argument |
| 0xA0 | Field whose low byte suppresses part of the broader update path |
| 0xAC | First captured comparison value |
| 0xB0 | Second captured comparison value |
| 0xB8 | Post-update field selecting the broader update path when nonzero |

These are access descriptions, not a complete object definition. In particular, this evidence does not establish the total object size or the full extent of the data beginning at offset 0x80.

### Associated lookup result

A successful lookup supplies an address from which the function reads three consecutive 32-bit words:

| Relative offset | Contract label |
|---|---|
| 0x00 | Requested scalar |
| 0x04 | First requested value |
| 0x08 | Second requested value |

The result is treated as readable storage. This function does not visibly release it.

### Output

The visible result is a 32-bit pointer-shaped value. Depending on the path, it is:

- Zero.
- The original associated-object address.
- The input object address.
- The second result from F-3f2c2a170c13.
- The result from F-6d16652f2d03.

No uniform ownership, validity lifetime, or success/failure interpretation follows from this return type.

### Preconditions

For defined execution of a selected path:

- The input object and every field actually accessed must be valid.
- The dispatch table and the selected entries must be callable.
- A nonzero lookup result must expose at least three readable 32-bit words.
- Every selected dependency must accept the supplied addresses and values.
- Temporary storage passed to dependencies must be sufficient under the actual calling convention.

There are no visible checks for a null input object, invalid dispatch entries, malformed lookup storage, or dependency failures other than the explicit zero-result checks described below.

## Behavior

### 1. Initial gating and lookup

The function first reads the associated-object address at offset 0x38.

- If it is zero, the function returns zero. It does not read the enable field for this decision and does not perform the lookup.
- If it is nonzero, the function reads the enable value at offset 0x44, replacing the third argument’s visible value.
- If that enable value is zero, the function returns the original associated-object address.
- Otherwise, it calls F-5710a8628cba with the associated-object address and an opaque static lookup argument.

If the lookup returns zero, the function returns zero without performing the property dispatch calls.

The lookup argument’s value, representation, and matching rules are unknown.

### 2. Capture and apply requested values

For a nonzero lookup result, the visible order is:

1. Read the object’s first comparison value at offset 0xAC.
2. Read its second comparison value at offset 0xB0.
3. Invoke the dispatch entry at table offset 0x114 with the object, capturing its 32-bit result as the previous scalar.
4. Read all three words from the lookup result.
5. Invoke the dispatch entry at table offset 0x11C with the object and the first requested value.
6. Invoke the dispatch entry at table offset 0x120 with the object and the second requested value.
7. Invoke the dispatch entry at table offset 0x118 with the object and the requested scalar.
8. Read the object field at offset 0xB8.

All three requested words are captured before the first setter-like dispatch call. Mutations of the lookup storage during those later calls therefore do not change the captured requested values.

The comparisons use the values captured before these setter-like calls. The function does not reread offsets 0xAC and 0xB0 to verify the resulting state.

Each dispatch-table access occurs at its respective call site. The evidence does not establish that an earlier callback preserves the object, its table address, or any global state.

### 3. Broader update path

This path is selected if any of the following is true:

- The field read at offset 0xB8 is nonzero.
- The captured first comparison value differs from the first requested value.
- The captured second comparison value differs from the second requested value.

Selection is based on 32-bit equality. It does not require that a setter actually changed the object.

Once this path is selected, the narrower path is not executed. The function eventually returns the input object address, regardless of whether the optional notifications below run.

The notification gate is evaluated in this order:

1. Call F-3fac671648b8.
2. If its result is nonzero, inspect the low byte of the object field at offset 0xA0.
3. If that byte is zero, read a byte through a global address.
4. Continue only if that global byte is nonzero.

The object-byte and global-byte reads are conditional; they are not both performed when an earlier gate fails.

If all gates pass:

1. Capture the related-object address from offset 0x34.
2. Call F-e57b05520b65 with a temporary destination and the address at object offset 0x80 supplied twice.
3. If the captured related-object address is nonzero, call F-ee8733a08210 with that temporary and an address 0xD0 bytes beyond the captured related-object address.
4. Call F-ffe0d92c57b7 with the input object.
5. If its result is nonzero, call F-c9b5d17d8d1e with that result and the temporary.
6. Then call F-4d037ecf0f82 with the same captured result and the same temporary.

The related-object address is captured before temporary construction. It is not reread afterward.

Likewise, the result of F-ffe0d92c57b7 is reused for both final calls. If the first call mutates or invalidates that result, the visible caller still supplies the captured value to the second call. Preservation and lifetime requirements remain dependency contracts, not guarantees established here.

### 4. Narrower region-update path

This path is considered only when:

- The field read at offset 0xB8 is zero.
- Both captured comparison values equal their corresponding requested values.

If the previous scalar also equals the requested scalar, the function returns zero. The three setter-like calls have already occurred.

If the scalar values differ:

1. Call F-960363c07b32 with the object and temporary output storage.
2. Copy four consecutive 32-bit words from that output storage into a separate working record.
3. Call F-3f2c2a170c13 with the object, the two captured comparison values, and the previous scalar.
4. Call F-3f2c2a170c13 with the object, the two requested values, and the requested scalar.
5. Compare the two returned 32-bit values for equality.

If the two results are equal, the function returns the second result and performs no transformation or final region-update call.

If they differ:

- Interpret the two results as signed 32-bit values for ordering.
- Replace the second word of the working record with the smaller result.
- Replace the fourth word with the larger result.
- Preserve the first and third copied words.
- Call F-d156b0b83eb3 with the working record, the object value at offset 0x84, and then the object value at offset 0x80.
- Call F-6d16652f2d03 with the object and that working record.
- Return the result of F-6d16652f2d03.

The transformation arguments are read after both calls to F-3f2c2a170c13. They are not snapshots taken before those calls. The supplied representation does not establish the relative evaluation order of the two field reads within the transformation call.

### Width and arithmetic rules

The visible model uses 32-bit words and pointer-shaped values.

- Equality checks compare complete 32-bit values, except the explicitly byte-sized gates.
- The field at offset 0xA0 is tested through its low byte; nonzero upper bytes alone do not suppress notification.
- The global gate tests a byte for zero versus nonzero.
- Ordering of the two F-3f2c2a170c13 results is signed 32-bit ordering, not unsigned address ordering.
- The related-object adjustment is a byte displacement of 0xD0. No bounds or overflow check is visible.
- No numerical interpretation of the other three-word values is established beyond equality and forwarding.

## Side effects

This function visibly initiates mutations through dispatch calls and dependencies rather than directly assigning persistent object fields.

Observable effects may include:

- Three ordered setter-like calls, even when all requested values equal the captured values.
- Optional construction and modification of a temporary.
- Two ordered notifications to a nonzero recipient.
- Modification of a four-word working record.
- A final region-update call.

There is no visible rollback. If a later dependency fails, traps, or terminates execution, earlier calls may already have changed state.

No explicit allocation or release operation is identified. Absence of an explicit release does not establish ownership: lookup storage, recipients, and returned values may be borrowed, retained, transferred, or otherwise managed by undocumented dependencies.

Temporary addresses are passed to callees. The visible caller uses them during the call sequence; it does not establish whether callees retain them or whether retaining them is permitted.

## Dependencies

| Opaque ID or dispatch offset | Established caller-side use |
|---|---|
| F-5710a8628cba | Lookup using the associated-object address and an opaque static argument; zero skips application |
| Dispatch 0x114 | Obtains the previous scalar |
| Dispatch 0x11C | Receives the first requested value |
| Dispatch 0x120 | Receives the second requested value |
| Dispatch 0x118 | Receives the requested scalar |
| F-3fac671648b8 | First gate for broader notifications |
| F-e57b05520b65 | Receives temporary destination and the same object-relative address twice |
| F-ee8733a08210 | Conditionally receives the temporary and related-object address plus 0xD0 |
| F-ffe0d92c57b7 | Produces the optional notification recipient |
| F-c9b5d17d8d1e | First notification using the captured recipient and temporary |
| F-4d037ecf0f82 | Second notification using those same captured values |
| F-960363c07b32 | Supplies temporary data from which four words are copied |
| F-3f2c2a170c13 | Called twice to obtain values used for equality and signed ordering |
| F-d156b0b83eb3 | Receives the working record and two current object-field values |
| F-6d16652f2d03 | Performs the final narrow-path operation; its result becomes the return value |

These descriptions do not guarantee purity, memory preservation, ownership behavior, error conventions, or particular domain semantics.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests. Unless stated otherwise, dependencies are assumed to return normally and provide valid storage.

1. **No associated object.** With offset 0x38 equal to zero, the result is zero. No lookup or setter-like dispatch occurs, irrespective of the third argument.

2. **Disabled associated object.** With a nonzero associated-object address and offset 0x44 equal to zero, the result is that associated-object address. The lookup is not called.

3. **Lookup miss.** With lookup enabled but F-5710a8628cba returning zero, the result is zero and none of the four dispatch entries is invoked.

4. **Unchanged values.** With a successful lookup, equal captured/requested values, and a zero post-update field at offset 0xB8, all three setter-like calls still occur. The result is zero, with neither broader notification nor narrow-region computation.

5. **Changed first value, failed notification gate.** A differing first comparison value selects the broader path. If F-3fac671648b8 returns zero, subsequent notification gates and calls are skipped, but the result remains the input object address.

6. **Byte-specific suppression.** If the field at offset 0xA0 has value 0x00000100, its low byte is zero, so that field does not suppress the notification sequence. The other gates must still pass.

7. **Post-update override.** Even when all three requested values match their captured counterparts, a nonzero field at offset 0xB8 selects the broader path and produces the input object address.

8. **Scalar change with equal derived results.** If only the scalar differs and both calls to F-3f2c2a170c13 return the same nonzero value, that value is returned. Neither F-d156b0b83eb3 nor F-6d16652f2d03 is called.

9. **Signed ordering boundary.** If the derived results are 0x80000000 and 0x00000001, the second working-record word becomes 0x80000000 and the fourth becomes 0x00000001. This follows signed ordering, despite the opposite unsigned ordering.

10. **Mutation between dependencies.** If either derived-value call changes offsets 0x80 or 0x84, the transformation receives the subsequently read values, not necessarily the values present before those calls.

## Uncertainties

The following remain **documented_only**:

- The domain meaning of the three lookup words and the three dispatch operations.
- The exact layout and semantics of the temporary used by broader notifications.
- Whether the four-word working record is a rectangle, range, or another structure.
- The identity and mutability rules of the global byte and static lookup argument.
- All dependency ownership, retention, failure, and exception behavior.
- Whether dependencies preserve the object, dispatch table, lookup storage, recipients, or globals.
- The full calling convention of F-3f2c2a170c13. The supplied representation shows temporary-address preparation around both calls, but does not reliably expose its role as an argument or other ABI requirement.
- The complete physical layout of local temporary storage. Reconstructed stack-storage annotations are not sufficient to establish allocation boundaries.
- Whether the pointer-shaped derived values are actual addresses or integer quantities represented through a pointer type.

No additional validation, retries, fallback behavior, or cleanup may be inferred from these unknowns.

## Evidence

This contract is based solely on the supplied static representation and dependency-ID mapping. The representation is evidence to assess, not an authoritative behavioral description in its comments or annotations.

Established evidence includes field offsets, visible call ordering, explicit branch conditions, 32-bit equality and signed ordering, conditional byte reads, the four-word copy and replacement pattern, and path-dependent return values.

Inferred intent is limited to the descriptive notions of setter-like dispatch, broader notification, and narrower region update. These labels do not replace the explicit conditions and call sequence.

No runtime tests have been performed. No dependency implementation, binary execution, complete ABI description, or independent object-layout definition was available.
