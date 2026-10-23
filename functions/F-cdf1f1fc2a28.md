<!-- Trusted provenance: F-cdf1f1fc2a28; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cdf1f1fc2a28

## Purpose

F-cdf1f1fc2a28 conditionally writes a supplied signed integer into a record obtained through an indirect call.

The decision uses a value returned by an earlier indirect call. Zero remains zero; every other returned value is reduced by one. If this adjusted value is greater than the supplied integer, the function requests a destination record and, if that request returns a non-null address, writes the supplied integer at byte offset 28 within that record.

This is the established control-flow behavior. Interpreting the compared value as a count, index, capacity, or position would be inferred intent, not an established contract.

## Interface

**Inputs**

1. **Object address.** An address used to access two indirect-call tables:
   - A related-object address is read from element 42 of an integer-sized array view.
   - A separate dispatch-table address is read from the beginning of the supplied object.
2. **Requested value.** A signed integer used both as the comparison operand and, conditionally, as the value written into the destination record.

**Output**

There is no explicit return value or status. Ordinary completion does not distinguish among:

- The comparison rejecting the update.
- The destination lookup returning null.
- A successful write.

Effects produced by dependencies may remain observable in all three cases.

**Width and layout model**

The supplied representation uses signed integer values and integer-held addresses consistent with a 32-bit model. Under that model:

| Location or value | Width or position |
|---|---|
| Requested value | Signed 32-bit integer |
| First indirect call’s result | Signed 32-bit integer |
| Destination address returned by the second indirect call | 32-bit address representation |
| Related-object field in the input object | Byte offset 168: element 42 × 4 bytes |
| First indirect-call entry | Byte offset 428 within its dispatch table |
| Second indirect-call entry | Byte offset 192 within its dispatch table |
| Destination field | Four bytes beginning at byte offset 28 |

The dispatch-table offsets and destination-field offset are explicit byte offsets. The conversion of element 42 to byte offset 168 depends on the four-byte integer model. The supplied material does not independently establish a complete target ABI, pointer width, alignment policy, or byte order.

**Preconditions for defined, ordinary completion**

The object and every subsequently dereferenced address must be valid for the accesses actually reached. In particular:

- The initial dependency must return normally.
- The related-object field must be readable after that dependency returns.
- The related object must expose a readable dispatch-table address at its beginning.
- The first dispatch table must contain a callable entry at byte offset 428.
- If the update branch is taken, the supplied object must expose a readable dispatch-table address at its beginning and a callable entry at byte offset 192.
- If the destination result is non-null, its field at byte offset 28 must be writable for an integer-sized store.

The function does not check these conditions. A null destination result is handled explicitly; null object, related-object, dispatch-table, and function-entry addresses are not.

## Behavior

### 1. Invoke the initial dependency

F-cdf1f1fc2a28 first invokes F-f7cb977a5079 with no explicit arguments.

No object-field read shown in this function precedes that invocation. Consequently, relevant object contents are obtained after the dependency returns, not from a snapshot captured before it.

The dependency’s return value is not used. Its other effects are unknown. It must not be assumed to preserve global state, leave the supplied object unchanged, or preserve allocations reachable from that object.

### 2. Obtain the comparison value

After the initial dependency returns, the function:

- Reads the related-object address from element 42 of the supplied object.
- Reads a dispatch-table address from the beginning of that related object.
- Reads the callable entry at byte offset 428 in that table.
- Invokes that entry, identified here as F-u01.

No explicit argument is supplied to F-u01 in the provided representation. In particular, the related-object address is used to locate the callable entry but is not visibly passed as an argument.

The returned value is interpreted as a signed integer for the subsequent adjustment and comparison.

### 3. Adjust and compare

If F-u01 returns zero, the comparison value is zero.

If F-u01 returns a nonzero value, the comparison value is that value minus one, subject to the arithmetic uncertainty for the minimum representable signed integer described below.

The function then performs a signed comparison:

- If the adjusted value is less than or equal to the requested value, the function returns immediately.
- If the adjusted value is greater than the requested value, the function proceeds to destination lookup.

Equality therefore suppresses the update. The condition is not “greater than or equal,” and the requested value is not clamped or normalized before comparison.

No nonnegativity requirement is enforced. Negative inputs and negative results participate in the signed comparison.

### 4. Obtain the destination

Only when the adjusted value is strictly greater than the requested value does the function:

- Read the dispatch-table address from the beginning of the supplied object.
- Read the callable entry at byte offset 192 in that table.
- Invoke that entry, identified here as F-u02, with the original supplied object address as its one explicit argument.

The dispatch-table read occurs after F-u01 returns. The function therefore uses the then-current table address rather than one captured before either dependency call.

F-u02’s result is treated as an address. Zero means no destination. A nonzero result permits the final store; there is no further validation.

### 5. Perform the conditional store

If F-u02 returns zero, the function returns without an explicit destination-field write.

Otherwise, the function stores the requested value at byte offset 28 from the returned address and then returns.

The stored value is the requested value, not the adjusted comparison value and not F-u01’s original result.

No destination-field read precedes the store. No equality check against the existing field value is performed. An already matching field is still written when this path is reached.

## Side effects

**Explicit effects**

The only explicit data write in F-cdf1f1fc2a28 is the conditional integer-sized store at destination byte offset 28. The function does not explicitly clear, initialize, copy, allocate, or release any other storage.

The visible ordering is:

1. F-f7cb977a5079 executes.
2. The related-object and first dispatch-table information are read.
3. F-u01 executes.
4. Adjustment and signed comparison occur.
5. If required, the supplied object’s dispatch-table information is read.
6. F-u02 executes.
7. If its result is non-null, the destination field is written.

This describes logical sequencing, not atomicity, memory barriers, or a hardware memory-order guarantee.

**Dependency effects**

Any dependency may have additional observable effects. A path with no explicit destination write is not necessarily side-effect-free.

No assumption is justified that F-u01 or F-u02 preserves globals, retains object lifetimes, avoids reentrancy, or leaves unrelated memory unchanged. Continued execution requires that memory needed for later operations remain valid.

**Ownership and lifetime**

No ownership transfer is established.

The function does not explicitly release either the supplied object or the returned destination. It does not store the destination address for later use. These observations do not establish whether F-u02 returns borrowed, owned, shared, or internally managed storage.

The returned destination must remain writable through the final store. Whether it aliases the supplied object, the related object, or other state is unknown.

**Errors**

There is no explicit error result, diagnostic, retry, or recovery mechanism.

A null destination is silently accepted as a no-write outcome. Invalid addresses, invalid callable entries, dependency failures, and arithmetic outside the established range have no defined recovery behavior in the supplied evidence.

## Dependencies

The following IDs describe dependencies without asserting unavailable implementation details.

| Opaque ID | Invocation and role | Established limits |
|---|---|---|
| F-f7cb977a5079 | Called first, unconditionally, with no explicit arguments | Return value unused; behavior and side effects unknown |
| F-u01 | Specification-local ID for the entry selected at byte offset 428 through the related object | Called once on ordinary execution; no explicit arguments shown; returns the integer used for comparison |
| F-u02 | Specification-local ID for the entry selected at byte offset 192 through the supplied object | Called only after the strict-greater comparison succeeds; receives the supplied object address; returns a nullable destination address |

F-u01 and F-u02 are opaque labels for call sites, not resolved function identities. Their actual targets may depend on the dispatch-table contents at the time of each read.

No implicit receiver, calling convention, ownership rule, or additional dependency contract is established.

## Acceptance examples

These examples are conditional on normal dependency returns, valid required memory, and the stated 32-bit signed interpretation. They describe expected behavior, not executed tests.

| F-u01 result | Requested value | Adjusted value | Required outcome |
|---:|---:|---:|---|
| 0 | 0 | 0 | Return without calling F-u02 |
| 1 | 0 | 0 | Return without calling F-u02 |
| 2 | 0 | 1 | Call F-u02; write 0 at destination offset 28 if its result is non-null |
| 6 | 5 | 5 | Return without calling F-u02 because equality suppresses the update |
| 6 | 4 | 5 | Call F-u02; write 4 if its result is non-null |
| 0 | −1 | 0 | Call F-u02; write −1 if its result is non-null |
| −1 | −2 | −2 | Return without calling F-u02 |
| −1 | −3 | −2 | Call F-u02; write −3 if its result is non-null |
| 2,147,483,647 | 2,147,483,645 | 2,147,483,646 | Call F-u02; write the requested value if its result is non-null |

Additional acceptance conditions:

- **Null destination:** When the comparison succeeds but F-u02 returns zero, no destination store occurs. Earlier dependency effects are not rolled back.
- **Existing matching value:** When the comparison succeeds and the destination is non-null, the store occurs even if the destination field already equals the requested value.
- **Mutation by the first dependency:** If F-f7cb977a5079 changes the related-object field and leaves it valid, the subsequent lookup uses the changed field.
- **Mutation by F-u01:** If F-u01 changes the supplied object’s dispatch-table address and the update branch is taken, F-u02 is selected using that changed address.
- **Zero normalization boundary:** Results of zero and one both produce a comparison value of zero. Zero is not reduced to −1.
- **Minimum signed result:** A result of −2,147,483,648 does not have a portable, established adjustment outcome in this contract.

## Uncertainties

**documented_only — Signed-minimum arithmetic.** Subtracting one from the minimum signed 32-bit value is outside the signed 32-bit range. The supplied representation does not establish whether the underlying execution wraps, traps, or has another consequence. No wrapping result is promised.

**documented_only — ABI and widths.** The four-byte integer and address model is consistent with the representation, but complete ABI details are absent. Alignment, endianness, exact callable-entry representation, and any hidden call parameters remain unspecified.

**documented_only — Dependency semantics.** The purposes of all three dependencies are unknown. In particular, the initial dependency cannot safely be classified as synchronization, initialization, or validation merely because it runs first.

**documented_only — Object meanings.** The meanings of the related-object field and destination field are not established. Structural offsets do not prove semantic names.

**documented_only — Lifetime and ownership.** No allocation, retention, borrowing, or release contract is available for dependency-returned storage.

**documented_only — Concurrency and abnormal completion.** No locking, atomicity, exception propagation, signal behavior, or concurrent-mutation guarantee is shown.

**Inferred intent.** The function appears to gate an update using a zero-preserving decrement of a dependency-provided value. Any stronger interpretation, including bounds enforcement or index selection, remains unproven.

## Evidence

The supplied static representation establishes the call order, field and dispatch-entry offsets, zero-preserving adjustment, signed comparison, conditional destination lookup, null-result branch, and final store.

The representation does not include dependency bodies, a complete ABI declaration, or runtime observations. Width-dependent layout calculations and signed-boundary limitations are therefore identified separately from directly visible control flow.

No runtime tests have been performed. The acceptance examples are conditional contract cases, not observed execution results.
