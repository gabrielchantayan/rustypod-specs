<!-- Trusted provenance: F-ee3bcf7c1950; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ee3bcf7c1950

## Purpose

F-ee3bcf7c1950 conditionally validates a signed position, resolves a distinguished “last position” request, and invokes an ordered sequence of object-associated operations.

Established behavior includes:

- An optional preliminary check using the original requested position.
- A special interpretation of the signed 32-bit value 2,147,483,647 when an object-associated signed count is positive.
- Validation through F-402eef64af83 for all other cases.
- An immediate return of −1 when that validation reports zero.
- For a selected position other than −1, an optional range operation, a mandatory indirect operation, and a final call to F-d6032bc0598f.
- Return of the selected position on the ordinary completion path.

The sequence is consistent with retaining a position while removing or otherwise processing later positions. That interpretation is inferred intent, not an established contract: the effects of the indirect operations and F-d6032bc0598f are unavailable.

## Interface

### Inputs

The function receives two arguments:

1. **Object reference.** A pointer to an object whose accessed representation contains:
   - At byte offset 0: a 32-bit dispatch-table address.
   - At byte offset 4: a signed 32-bit quantity, called the **count** in this contract.
   - At byte offset 40, hexadecimal 0x28: a 32-bit word whose conversion to a character-sized value controls the preliminary check.

2. **Requested position.** A signed 32-bit integer.

“Count” and “position” describe how these values are used, not independently established domain constraints. The function does not itself establish that the count is nonnegative or that the requested position denotes a valid element.

The object reference and its dispatch table must support every access reached on the selected execution path. No null-object check, object-size check, dispatch-table check, or indirect-target check is visible.

### Output

On ordinary completion, the function returns a signed 32-bit integer:

- −1 if F-402eef64af83 returns zero.
- Otherwise, the selected position, which is either the original requested position or the count minus one selected by the distinguished-value branch.

A result of −1 is therefore ambiguous: it can indicate validation failure or ordinary completion with a selected position of −1.

A separate preliminary-check path invokes F-feee9156a257, which is statically marked as non-returning. No ordinary return is established for that path.

### Representation and arithmetic

The supplied representation uses four-byte integer words, 32-bit address-bearing fields, and signed 32-bit position arithmetic.

The preliminary flag test narrows the word at byte offset 40 to a character-sized value before testing for zero. Under the represented 8-bit-character model, only its low eight bits determine this test. A nonzero 32-bit word whose low byte is zero therefore does not activate the preliminary check.

The routine uses two distinct all-one values:

- Signed −1 as a position or return value.
- The 32-bit argument pattern 0xFFFFFFFF in indirect calls.

These have the same bits in the represented model, but the semantic interpretation of the indirect-call arguments is unknown.

## Behavior

### 1. Optional preliminary check

The function first reads and narrows the word at object byte offset 40.

If the narrowed value is zero, it skips the preliminary indirect call and proceeds to position selection.

If the narrowed value is nonzero, it:

1. Reads the object’s dispatch-table address.
2. Selects the indirect target at dispatch-table byte offset 0x40.
3. Calls that target with the object reference and the original requested position.
4. Treats the result as a pointer to a 32-bit integer.
5. Reads the integer at that returned address.

There is no visible null check on the returned pointer.

If the returned integer is zero, processing continues. If it is nonzero, the function calls F-588f8c1c803a and, if that call returns, calls F-feee9156a257. The latter is marked non-returning in the supplied static evidence.

The preliminary check precedes both the distinguished-value branch and ordinary validation. Consequently, it receives the original requested value even when that value will later be replaced by count minus one.

No count snapshot is taken before this indirect call. Changes made by it to the object can affect subsequent selection and dispatch.

### 2. Position selection and validation

After any returning preliminary check, the function examines the requested position.

If it equals 2,147,483,647 and the current signed count is greater than zero:

- The selected position becomes the current count minus one.
- F-402eef64af83 is not called.

Otherwise:

- F-402eef64af83 is called with the object reference and the original requested position.
- A zero result causes an immediate return of −1.
- Any nonzero result allows processing to continue with the requested position unchanged.

The distinguished value is not unconditionally converted. With a zero or negative count, it goes through F-402eef64af83 like other requests.

The count-minus-one operation in the distinguished-value branch is within the signed 32-bit range because that branch requires a positive count. Its selected position is therefore between 0 and 2,147,483,646 inclusive.

Validation is delegated. The visible function does not independently reject negative positions, positions beyond the count, or −1.

### 3. Selected position of −1

After successful selection or validation, a selected position of −1 suppresses all remaining operations.

The function returns −1 without:

- Calling the indirect target at dispatch-table offset 0xB8.
- Calling the indirect target at dispatch-table offset 0xBC.
- Calling F-d6032bc0598f.

This is not the same execution path as validation failure. A request of −1 can reach this branch only after F-402eef64af83 has returned nonzero, and the preliminary check may already have occurred.

### 4. Optional range operation

For a selected position other than −1, the function compares the selected position plus one with the current signed count.

If the former is strictly less than the latter, it invokes the indirect target at dispatch-table byte offset 0xB8. The visible argument values are:

- The object reference.
- Selected position plus one.
- Current count minus one.
- The 32-bit pattern 0xFFFFFFFF.
- Four additional 32-bit values originating from ambient machine state rather than either explicit input.

The position expressions suggest endpoints for a suffix following the selected position. Whether the indirect dependency treats those endpoints as inclusive, and what it does with the supplied values, remain unknown.

If the comparison is false, this indirect operation is skipped.

The comparison and endpoint construction use the object state available after F-402eef64af83, if validation was called. A count from before validation is not preserved for this stage.

### 5. Mandatory indirect operation and final dependency

For every selected position other than −1, after the optional range operation returns or is skipped, the function:

1. Selects the indirect target at dispatch-table byte offset 0xBC.
2. Calls it with the object reference and 0xFFFFFFFF.
3. Calls F-d6032bc0598f with the object reference and selected position.
4. Returns the selected position.

The offset-0xBC target is selected after the optional offset-0xB8 call. The function does not require those two calls to use the same dispatch-table address: an earlier dependency may change the object’s dispatch-table field.

The visible results of these final calls do not determine the returned position. Their side effects, failures, and non-returning behavior can nevertheless affect whether ordinary completion is reached.

### 6. Signed arithmetic boundaries

Signed overflow behavior is not established by the supplied static representation. Two cases require particular caution:

- A validated selected position of 2,147,483,647 reaches an addition of one during the range comparison.
- A range call reached with a count of −2,147,483,648 requires a count-minus-one endpoint.

The contract does not assign wrapping, trapping, or other behavior to these overflow cases. They are documented_only.

Other negative positions are not intrinsically rejected by this routine. For example, a validated selected position of −2 can proceed into the range and final-operation sequence.

## Side effects

The supplied body contains no explicit store to object fields or global memory. Its observable side effects are delegated to dependencies.

Those dependencies may modify the object, other memory, global state, resources, or control flow. No preservation guarantee is available for them.

Relevant observable ordering is:

1. Preliminary flag read.
2. Optional offset-0x40 indirect call and returned-word read.
3. Optional F-588f8c1c803a and F-feee9156a257 calls.
4. Distinguished-value selection or F-402eef64af83 validation.
5. Conditional range comparison and offset-0xB8 indirect call.
6. Offset-0xBC indirect call.
7. F-d6032bc0598f.
8. Ordinary return.

The count is not frozen across dependency calls. Likewise, later indirect targets are obtained from object state reached at their respective call sites.

No allocation, release, reference-count operation, or ownership transfer is explicit. In particular:

- The object reference is passed to dependencies, but ownership transfer is not established.
- The pointer returned by the offset-0x40 dependency is dereferenced but not explicitly released.
- The absence of a release here does not prove that the returned storage is borrowed, permanent, or owned elsewhere.
- No assumption is made that unknown callees preserve global state or keep the object alive.

Object lifetime and pointer validity must remain sufficient for subsequent accesses unless a dependency does not return.

## Dependencies

| Dependency | Invocation condition | Established local use |
|---|---|---|
| F-588f8c1c803a | Preliminary indirect result points to a nonzero word | Called with no explicit arguments; no result is used. |
| F-feee9156a257 | Same preliminary branch, if F-588f8c1c803a returns | Called with no explicit arguments; marked non-returning. |
| F-402eef64af83 | Distinguished-value selection is not taken | Receives object and requested position; zero rejects, nonzero accepts continuation. |
| F-d6032bc0598f | Selected position is not −1 and prior operations return | Receives object and selected position; no result is used. |
| Unresolved indirect dependency at offset 0x40 | Narrowed preliminary flag is nonzero | Receives object and original request; supplies a pointer whose first 32-bit word is tested. |
| Unresolved indirect dependency at offset 0xB8 | Selected position is not −1 and its successor is below current count | Receives the range-related values and additional ambient values described above. |
| Unresolved indirect dependency at offset 0xBC | Selected position is not −1 | Receives object and 0xFFFFFFFF. |

The dispatch offsets identify selection locations, not stable function identities. The indirect implementations and their contracts are unavailable.

## Acceptance examples

These are conditional static acceptance examples, not executed tests. Unless stated otherwise, assume valid memory, returning dependencies, no relevant object mutation by dependencies, and representable arithmetic.

### Distinguished request with a positive count

Given a requested position of 2,147,483,647 and count 5:

- The selected position becomes 4.
- F-402eef64af83 is skipped.
- The range operation is skipped because 5 is not less than 5.
- The offset-0xBC dependency is called.
- F-d6032bc0598f receives position 4.
- The return value is 4.

If the preliminary flag is enabled, its indirect dependency first receives 2,147,483,647, not 4.

### Accepted position with a following range

Given requested position 1, count 5, and a nonzero validation result:

- The offset-0xB8 call receives range-related positions 2 and 4.
- The offset-0xBC call follows it.
- F-d6032bc0598f receives position 1.
- The return value is 1.

This establishes argument values and order, not what the range call changes.

### Accepted final position

Given requested position 4, count 5, and a nonzero validation result:

- The range call is skipped.
- The offset-0xBC call and F-d6032bc0598f still occur.
- The return value is 4.

### Validation rejection

Given any request that reaches F-402eef64af83, if that dependency returns zero:

- The return value is −1.
- Neither final indirect operation nor F-d6032bc0598f is called.
- Any preliminary-check effects have already occurred.

### Accepted request of −1

Given requested position −1 and a nonzero validation result:

- Validation occurs.
- All three post-validation operations are skipped.
- The return value is −1.

### Validation changes the count

Given requested position 1 and an initial count of 5, suppose validation returns nonzero but changes the count to 2:

- The later comparison uses count 2.
- The range call is skipped because 2 is not less than 2.
- The remaining two operations still occur.

### Preliminary condition selects the non-returning path

Given an enabled preliminary flag and a valid returned pointer to a nonzero word:

- F-588f8c1c803a is called.
- If it returns, F-feee9156a257 is called.
- No ordinary return or subsequent validation is established.

### Distinguished request with a nonpositive count

Given the distinguished request and count zero or negative:

- No count-minus-one selection occurs.
- F-402eef64af83 is called.
- If it returns zero, the function returns −1.
- If it returns nonzero, subsequent successor arithmetic reaches an unresolved overflow boundary; no wrapping result is specified.

## Uncertainties

The following remain documented_only:

- Domain meaning and valid ranges of the object fields.
- Full object layout and dispatch-table layout.
- Concrete identities and contracts of the indirect dependencies.
- Ownership and lifetime of the preliminary result pointer.
- Semantics of the range-related arguments and 0xFFFFFFFF.
- Meanings and reliability of the four ambient values supplied to the offset-0xB8 call.
- Exact machine behavior on signed overflow.
- Whether dependencies mutate counts, replace dispatch tables, release resources, throw, terminate, or otherwise fail to return.
- Concurrency behavior and synchronization requirements.
- Mechanism of the non-returning dependency.

The apparent position-processing intent must not be promoted into guarantees about deletion, truncation, notification, or resource release.

## Evidence

This contract is based solely on the supplied static function representation and dependency-ID mapping.

Established evidence covers branch conditions, accessed offsets, visible argument values, signed comparisons, call ordering, ignored results, ordinary returns, and the non-returning annotation. Semantic interpretations are identified as inferred or unknown.

No implementation access, runtime execution, or runtime tests were performed.
