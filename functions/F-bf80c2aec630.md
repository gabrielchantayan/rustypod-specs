<!-- Trusted provenance: F-bf80c2aec630; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bf80c2aec630

## Purpose

F-bf80c2aec630 performs an ordered sequence of object dispatch, a fixed-size transfer-like operation, and optional validation.

The established behavior is:

1. Invoke an operation on an object referenced by the primary object, passing a supplied selector.
2. Invoke an external operation with a destination inside the primary object, a selector-derived source address, and a size of 20 bytes.
3. Obtain an optional secondary object through the primary object.
4. If a secondary object exists, obtain a value from it, substitute a global fallback when that value is zero, and pass the resulting value to another operation on the primary object.
5. If that final operation returns zero, invoke F-a817d8d58c10, which is marked non-returning.

The external operation’s arguments suggest a fixed-size record copy, but its implementation is unavailable. Likewise, “selector,” “secondary object,” and “validation” are descriptive interpretations, not established domain meanings.

## Interface

### Inputs

The function accepts two arguments:

- **Primary object:** a pointer to an integer-addressed object representation. The function uses this object both as a dispatch receiver and as the base of an inline destination region.
- **Selector:** an integer passed unchanged to the first dispatched operation and used to derive the source address for the external operation.

The supplied representation uses 32-bit integer and pointer-sized entries. Under that representation, the relevant byte offsets are:

| Location | Role |
|---|---|
| Primary object + 0 | Dispatch-table pointer for the primary object |
| Primary object + 64 | Pointer to the object used by the first dispatch |
| Primary object + 72 | Start of the 20-byte destination region |
| First referenced object + 0 | Dispatch-table pointer for that object |
| Secondary object + 0 | Dispatch-table pointer for the secondary object |

The destination region therefore covers byte offsets 72 through 91 of the primary object, inclusive.

The dispatch-table entries used are:

| Receiver or table owner | Entry byte offset | Explicit arguments |
|---|---:|---|
| Object referenced at primary-object offset 64 | 44 | Referenced object pointer, selector |
| Primary object | 100 | Primary object pointer |
| Secondary object | 4 | None shown |
| Primary object | 64 | Primary object pointer, selected value |

These offsets describe the supplied 32-bit representation, not a portable layout for a different pointer width.

### Output

There is no explicit return value.

There are two established ordinary-return paths:

- The primary-object operation at dispatch-table offset 100 returns a null pointer.
- A non-null secondary object is obtained, and the final primary-object operation returns a nonzero integer.

A third path invokes the non-returning dependency F-a817d8d58c10. No ordinary return is established for that path.

### Preconditions

The function performs no explicit argument, bounds, or pointer validation before using its inputs.

For execution to proceed through the applicable path:

- The primary object must support the reads, dispatches, and destination addressing described here.
- The pointer at primary-object offset 64 must identify an object with a usable dispatch entry at byte offset 44.
- The primary object must have a usable dispatch entry at byte offset 100.
- If a secondary object is returned, it must support reading its dispatch-table pointer and calling the entry at byte offset 4.
- If execution reaches the final primary-object dispatch, the primary object must still have a usable dispatch entry at byte offset 64.
- The source address derived from the selector must satisfy the external operation’s actual requirements.

The supplied evidence does not establish whether invalid pointers, invalid selectors, inaccessible memory, overlapping regions, or corrupted dispatch tables are detected by any dependency.

## Behavior

### 1. Invoke the referenced object’s operation

The function obtains an object pointer from byte offset 64 of the primary object. Through that object’s dispatch table, it invokes the entry at byte offset 44.

The explicit arguments are the referenced object pointer and the selector.

No result from this call is used by the enclosing function. The call must finish before the external transfer-like operation is invoked.

The dependency may change memory or global state. The supplied evidence does not establish that it preserves the primary object, its dispatch table, the global source base, or any later input. Consequently, subsequent operations must be understood as using the state available after this call, not an assumed snapshot from function entry.

### 2. Invoke the external operation on a 20-byte region

After the first dispatched operation returns, the function invokes an external operation with three arguments:

1. Destination: primary-object address plus 72 bytes.
2. Source: a global base value plus the selector multiplied by 20.
3. Size: 20.

The stride and size are both 20 bytes. The selector is not visibly constrained to any particular range, and there is no explicit check for a negative value.

The arithmetic expression uses an integer selector and an integer stride. The supplied evidence does not establish a safe selector range, the signed-overflow treatment of the original executable, or the validity of any wrapped address. A contract consumer must not infer that all integer selector values produce valid source addresses.

The call is consistent with copying one 20-byte table record into inline object storage. That interpretation remains conditional on the external operation’s identity and semantics. The direct evidence establishes its invocation and argument values, not its memory effects.

No return value from this external operation is used.

### 3. Obtain an optional secondary object

After the external operation returns, the function uses the primary object’s current dispatch table to invoke the entry at byte offset 100, passing the primary object pointer.

The returned value is interpreted as a pointer.

If it is null, the function returns immediately. On this path:

- The earlier referenced-object dispatch has already occurred.
- The external operation has already been invoked.
- No secondary-object dispatch occurs.
- No fallback substitution occurs.
- No final primary-object dispatch occurs.
- F-a817d8d58c10 is not invoked by this function.

The null result is therefore a normal early-return condition, not a reason to skip the preceding operations.

### 4. Obtain a value from the secondary object

If the secondary-object pointer is non-null, the function reads its dispatch-table pointer and invokes the entry at byte offset 4.

No explicit arguments are shown for this call. In particular, the supplied evidence does not show the secondary-object pointer being passed as an explicit receiver argument. A behavioral contract must not silently add one.

The operation returns an integer value.

- If that value is nonzero, the function uses it unchanged.
- If that value is zero, the function reads a global fallback value and uses that instead.

The fallback is selected solely by comparison with zero. Negative values are nonzero and therefore do not trigger substitution.

The fallback itself is not checked for zero. A fallback value of zero is passed onward as zero.

### 5. Invoke the final primary-object operation

The function uses the primary object’s dispatch table to invoke the entry at byte offset 64. The explicit arguments are:

1. The primary object pointer.
2. The value selected in the preceding step.

This dispatch occurs after the secondary-object call and any fallback read. The supplied evidence does not establish that those earlier operations preserve the primary object or its dispatch table.

The result is compared with zero:

- **Nonzero:** return normally.
- **Zero:** invoke F-a817d8d58c10.

Any nonzero result satisfies the return condition; the function does not require a particular positive success code.

### 6. Invoke the non-returning dependency on zero

If the final dispatched operation returns zero, F-a817d8d58c10 is invoked without explicit arguments.

The dependency is marked non-returning in the supplied evidence. Its mechanism is unknown: process termination, an exception-like transfer, or another form of control transfer is not established.

There is no visible cleanup, retry, rollback, or alternate result handling between the zero result and this invocation.

## Side effects

The observable ordering is:

1. Referenced-object dispatch.
2. External operation with the 20-byte destination and selector-derived source.
3. Primary-object dispatch that obtains the optional secondary object.
4. If non-null, secondary-object dispatch.
5. If its result is zero, global fallback read.
6. Final primary-object dispatch.
7. If its result is zero, invocation of F-a817d8d58c10.

Later stages are conditional on earlier calls returning and the stated branch conditions.

The enclosing function visibly performs dispatch-related memory reads and reads global values. It does not contain an explicit allocation, deallocation, ownership transfer, or release operation.

The external operation may write the destination region, and any dependency may have additional side effects. Those effects cannot be bounded from the supplied body alone. In particular:

- The 20-byte size argument is not proof that no other memory is touched.
- A secondary-object pointer is used but not visibly released.
- No ownership of that pointer is established.
- Normal return does not imply that earlier state changes were undone.
- The non-returning path occurs after all preceding applicable calls; no rollback is shown.

The evidence establishes call order, but not a complete ordering of every machine-level load used to evaluate dispatch targets and arguments.

## Dependencies

Opaque identifiers below distinguish unavailable call targets; they do not assert globally assigned identities for dynamically selected methods.

| Opaque ID | Established role | Unknown aspects |
|---|---|---|
| D-01 | Dispatch through the referenced object’s table at byte offset 44; receives that object and the selector | State changes, failure behavior, actual target identity |
| D-02 | External operation receiving destination, selector-derived source, and size 20 | Whether it copies, overlap behavior, access requirements, other effects |
| D-03 | Primary-object dispatch at table byte offset 100; returns an optional pointer | Ownership, object lifetime, state changes |
| D-04 | Secondary-object dispatch at table byte offset 4; no explicit arguments; returns an integer | Calling convention details, meaning of the value, state changes |
| D-05 | Primary-object dispatch at table byte offset 64; receives the selected value; returns an integer tested against zero | Meaning of success or failure, state changes |
| F-a817d8d58c10 | Invoked when D-05 returns zero; marked non-returning | Termination or control-transfer mechanism, cleanup, diagnostics |

None of these dependencies is assumed to preserve globals, preserve object contents, or acquire or release ownership.

## Acceptance examples

These are conditional behavioral examples derived from static evidence. They are not runtime test results.

### Null secondary object

Given valid dispatches and a returning D-02, if D-03 returns null:

- D-01 and D-02 occur first.
- The function then returns.
- D-04, D-05, and F-a817d8d58c10 are not invoked.

### Nonzero secondary value

If D-03 returns a usable non-null pointer, D-04 returns 7, and D-05 returns 1:

- D-05 receives the primary object and 7.
- The global fallback is not selected.
- The function returns normally.

### Negative secondary value

If D-04 returns −1, that value is passed unchanged to D-05. It does not trigger fallback substitution.

### Zero secondary value with a nonzero fallback

If D-04 returns zero and the fallback read produces 12, D-05 receives 12 as its second explicit argument.

### Zero secondary value with a zero fallback

If D-04 returns zero and the fallback is also zero, D-05 receives zero. The function does not reject that value before the call.

### Final zero result

If D-05 returns zero, F-a817d8d58c10 is invoked. No ordinary return is established, regardless of whether the selected input value was zero or nonzero.

### Selector-derived addresses

Assuming arithmetic is representable and the addresses meet D-02’s requirements:

- Selector 0 produces the global source base.
- Selector 1 produces the base plus 20 bytes.
- Selector −1 produces the base minus 20 bytes.

The function does not visibly reject the negative-selector case. Its validity remains dependent on the actual source layout and external-operation contract.

## Uncertainties

The following are **documented_only**:

- The external operation’s actual semantics, including whether it is a memory-copy operation.
- Source-table extent, record meaning, valid selector range, and overlap rules.
- Exact overflow and address-wrap behavior.
- Domain meanings of the dispatched operations and their return values.
- Calling-convention details beyond the explicit arguments shown.
- Object ownership and lifetime guarantees.
- Dependency effects on globals, dispatch tables, and object storage.
- Whether invalid memory or invalid dispatch targets produce diagnostics, termination, or other behavior.
- The mechanism and side effects of F-a817d8d58c10.
- Exact machine-load counts and operand-evaluation ordering within individual call expressions.

Inferred intent is limited to a selector-driven record transfer followed by optional validation. It does not override the established call sequence or fill in missing dependency contracts.

## Evidence

The contract is based solely on the supplied static function body and its dependency mapping.

Established evidence includes the dispatch offsets, explicit arguments, 20-byte size and stride, null and zero comparisons, conditional fallback read, ordinary-return branches, and final invocation of F-a817d8d58c10.

The non-returning status of F-a817d8d58c10 comes from the supplied annotation rather than an inspected implementation.

No runtime tests have been performed. No dependency implementation has been inspected.
