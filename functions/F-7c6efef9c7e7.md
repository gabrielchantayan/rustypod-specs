<!-- Trusted provenance: F-7c6efef9c7e7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7c6efef9c7e7

## Purpose

F-7c6efef9c7e7 conditionally obtains a nonzero lookup result, passes it through one of two construction dependencies, and records the resulting 32-bit value in a caller-supplied object. It then marks that object with a one-byte completion value.

The routine recognizes two selector values supplied by global storage. One selector permits a direct primary lookup. The other permits a secondary lookup, with a primary lookup as fallback if the secondary lookup returns zero. Other selector values cause an early return.

The apparent intent is one-time, selector-dependent initialization. That interpretation is inferred from the completion check and the final writes. The established behavior is narrower: the routine checks a byte, performs conditional calls, stores a returned word, and writes the byte to one. The source does not establish what the constructed value represents, whether it is valid when zero, or who owns it.

## Interface

**Input:** one address-bearing argument identifying the base of a mutable object.

**Output:** no explicit return value. Results are observable through dependency calls and changes to object memory.

The supplied representation uses a 32-bit integer argument for the object address, 32-bit words for selectors and dependency results, and individual bytes for the completion state. Treating this as a native pointer interface on a wider-address machine is not established.

### Object layout

Offsets are relative to the supplied object base.

| Offset | Width or role | Observed use |
|---|---|---|
| `0x38` | 32-bit word | First argument to lookup calls |
| `0x40` | 32-bit word | Compared with the two global selectors |
| `0x44` | 32-bit word | Third argument to lookup calls |
| `0xA1` | One byte | Early-return guard; later overwritten with `1` |
| `0xA4` | Address of an interior region | Passed to the primary construction dependency |
| `0x15C` | 32-bit word | Receives the selected construction dependency’s result |

The two adjacent regions around `0xA1` and `0xA4` have distinct observed roles. The byte at `0xA1` is directly read and written. The address at offset `0xA4` is passed onward without any direct inspection of that region by this routine.

Two global 32-bit selector values participate in selection. This contract calls them the **primary selector** and the **secondary selector**. These are descriptive labels, not claims about their semantic domain.

### Preconditions

On any executed path, the memory directly accessed by that path must be valid for the corresponding read or write. There is no explicit null check, range check, alignment check, or object-size check.

A contiguous object extending through offset `0x15F` would cover all direct object accesses. That is not a sufficient description of dependency requirements: the primary construction dependency receives an interior address and may require additional memory or invariants not visible here.

Address additions are expressed through a 32-bit address-bearing representation. Overflow handling, address wraparound, and the behavior of invalid or misaligned accesses are not established. A portable implementation must not invent defined wraparound or safe failure behavior for those cases.

## Behavior

### 1. Capture the primary selector and check completion

The routine first reads the primary selector from global storage and retains that value for a possible primary lookup.

It then reads the byte at object offset `0xA1`.

- If the byte is nonzero, the routine returns immediately.
- Any nonzero byte satisfies this guard, not only `1`.
- This path makes no dependency calls and performs no direct writes to the object.
- The initial global read still precedes the guard.

Consequently, “already complete” does not mean that the routine performs no reads at all. It still reads the primary selector and the completion byte.

### 2. Select a lookup path

If the completion byte is zero, the routine compares the 32-bit selector field at offset `0x40` with the primary selector read from global storage for that comparison.

If they match, execution proceeds to the primary lookup.

If they do not match, the routine performs another comparison of the selector field at offset `0x40`, this time against the secondary selector from global storage.

- If this second comparison also fails, the routine returns without dependency calls or direct object writes.
- If it succeeds, execution first attempts the secondary lookup.

The supplied representation contains separate accesses for the initial primary-selector capture and its later comparison, and separate selector-field accesses for the two comparisons. This contract does not collapse them into one immutable snapshot. Conversely, it does not establish volatile-access semantics, atomicity, or a machine-level memory-order guarantee.

If the primary and secondary selectors have equal values and the observed object selector matches that value, the primary comparison wins. The secondary lookup is not attempted.

### 3. Attempt the secondary lookup when selected

The secondary lookup calls F-5710a8628cba with:

1. The 32-bit word read from object offset `0x38`.
2. The secondary selector read from global storage for this call.
3. The 32-bit word read from object offset `0x44`.

The source does not establish the evaluation order among these three argument reads. The call occurs after the successful secondary comparison, but the comparison does not establish that the same secondary-selector value must be observed again if global storage can change.

The returned 32-bit value is tested against zero.

**Nonzero result:**

- Call F-8157a180d901 with the numeric argument `0x44`, or 68.
- Pass its returned 32-bit value, together with the nonzero lookup result, to F-1ade66241a82.
- Use the return value of F-1ade66241a82 as the value to record.
- Skip the primary lookup and primary construction dependency.
- Proceed to the final writes.

**Zero result:**

- Proceed to the primary lookup.
- Do not call F-8157a180d901 or F-1ade66241a82 for this unsuccessful secondary attempt.

### 4. Perform the primary lookup

The primary lookup is reached either directly from a matching primary selector or after a zero secondary lookup result.

It calls F-5710a8628cba with:

1. The 32-bit word read from object offset `0x38`.
2. The primary-selector value captured at routine entry.
3. The 32-bit word read from object offset `0x44`.

The primary-selector argument is the retained entry value. It is not refreshed after a secondary lookup.

The object words used for this lookup are read for this call. If a preceding secondary lookup changes those words, the primary lookup can therefore receive changed values. The routine does not repeat the completion check or selector comparisons before falling back.

The source does not establish the evaluation order among this call’s argument reads.

If the primary lookup returns zero, the routine returns without making any subsequent direct object writes or construction calls.

If the primary lookup returns nonzero:

- Call F-8157a180d901 with `0x44`.
- Pass its returned 32-bit value, the address at object offset `0xA4`, and the nonzero primary lookup result to F-f8beff1830ad.
- Use the return value of F-f8beff1830ad as the value to record.
- Proceed to the final writes.

### 5. Record the result and mark completion

After either construction path returns normally, the routine performs these direct writes in order:

1. Store the construction dependency’s 32-bit return value at object offset `0x15C`.
2. Store the one-byte value `1` at object offset `0xA1`.

The construction result is not checked for zero. A zero result is stored and still followed by the completion write.

The previous word at offset `0x15C` is neither inspected nor explicitly released by this routine. The final store overwrites it regardless of its previous value, provided execution reaches this point.

These writes are sequential in the supplied representation, not a transactional update. No synchronization, atomic publication, or rollback mechanism is shown.

## Side effects

The routine’s direct writes are limited to the 32-bit result slot at `0x15C` and the completion byte at `0xA1`. Both occur only after a construction dependency returns normally.

Early-return paths do not directly clear, initialize, or otherwise modify either location. In particular:

- An unsupported selector leaves an existing result word unchanged.
- A zero primary lookup result leaves an existing result word unchanged.
- A failed secondary lookup followed by a failed primary lookup does not itself change the completion byte.

These are statements about this routine’s direct actions, not guarantees that the entire call leaves memory unchanged. Dependencies may mutate the object, global selectors, or other state. Their effects are unspecified here.

There is no explicit release, destruction, logging, retry loop, or error-reporting operation. At most two lookup calls and one call to each dependency in the selected construction sequence occur during a normal invocation.

No ownership transfer can be established. F-8157a180d901 resembles a resource-acquisition operation because it receives 68 and its result is passed to a construction dependency, but neither allocation semantics nor a 68-byte allocation is proven. Its result is not checked before being passed onward.

If a dependency does not return normally, subsequent actions in the described sequence are not reached. No cleanup or recovery path is visible.

## Dependencies

| Opaque function ID | Arguments supplied here | Result handling |
|---|---|---|
| F-5710a8628cba | Three 32-bit words: object lookup input, selected global-selector value, object lookup input | Zero means this attempt did not produce a construction input; nonzero is passed to a construction dependency |
| F-8157a180d901 | Numeric value `0x44` | Returned 32-bit value is forwarded without validation |
| F-1ade66241a82 | Result from F-8157a180d901; nonzero secondary lookup result | Returned 32-bit value is stored in the object |
| F-f8beff1830ad | Result from F-8157a180d901; object-base-plus-`0xA4` address; nonzero primary lookup result | Returned 32-bit value is stored in the object |

The names “lookup” and “construction” describe this routine’s use of these dependencies. They do not establish the dependencies’ internal algorithms, memory ownership, failure conventions beyond the caller’s observed zero tests, or preservation of global state.

## Acceptance examples

These examples are conditional behavioral checks, not reports of executed tests. Unless stated otherwise, assume valid memory, normal dependency returns, and no dependency or external actor changing the relevant object fields or globals.

1. **Completion byte is `2`.**  
   The routine reads the entry primary selector and the completion byte, then returns. It makes no dependency calls and does not replace `2` with `1`.

2. **Completion byte is zero; object selector matches neither selector.**  
   The routine returns after the comparisons. The result word and completion byte receive no direct writes.

3. **Object selector matches the primary selector; lookup returns zero.**  
   One call to F-5710a8628cba occurs with the entry-captured primary selector. No construction sequence or final writes occur.

4. **Primary lookup succeeds; primary construction returns zero.**  
   F-8157a180d901 receives 68. F-f8beff1830ad receives its result, the interior address at offset `0xA4`, and the nonzero lookup result. The routine stores zero at `0x15C`, then stores one at `0xA1`.

5. **Object selector matches only the secondary selector; secondary lookup succeeds.**  
   The routine uses F-1ade66241a82 after F-8157a180d901. It does not attempt a primary lookup and does not call F-f8beff1830ad. It records the secondary construction result and then marks completion.

6. **Secondary lookup returns zero; primary lookup succeeds.**  
   Two lookup calls occur in secondary-then-primary order. Only the primary construction sequence occurs. The fallback uses the entry-captured primary selector.

7. **Secondary lookup changes the object’s lookup inputs and then returns zero.**  
   The primary lookup reads the words at `0x38` and `0x44` after that dependency returns. It is not required to receive their pre-secondary-call values. No completion recheck intervenes.

8. **F-8157a180d901 returns zero.**  
   Zero is still supplied as the first argument to the selected construction dependency. This routine does not treat it as an early-return condition.

9. **Both global selectors equal the matching object selector.**  
   With stable values, the primary branch is selected. Equality of the selectors does not cause two lookup attempts.

## Uncertainties

The following remain **documented_only**:

- The semantic meaning and valid domains of the selector values and lookup inputs.
- Whether any returned word is a pointer, handle, status, or another representation.
- Whether F-8157a180d901 acquires memory or another resource, and what its argument measures.
- Ownership, lifetime, release obligations, and mutation behavior of all dependency arguments and results.
- The extent and required contents of the region beginning at offset `0xA4`.
- Dependency behavior on zero resource results, invalid inputs, or internal failure.
- Global preservation, callbacks, reentrancy, concurrent mutation, and thread safety.
- Endianness, exact calling convention, machine-level access ordering, and fault behavior.
- Whether the completion byte is intended as a lock or publication flag. The shown operations do not establish either guarantee.

## Evidence

Established static evidence consists of the supplied single routine: its explicit offsets, byte and word accesses, zero comparisons, selector-dependent branches, dependency arguments, fallback sequence, and final write order.

The one-time initialization interpretation and resource/construction terminology are inferred intent, not independently established semantics.

No dependency implementations or external interface definitions were supplied. No runtime tests have been performed.
