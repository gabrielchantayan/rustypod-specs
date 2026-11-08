<!-- Trusted provenance: F-fdb6d4e0b2c8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fdb6d4e0b2c8

## Purpose

F-fdb6d4e0b2c8 conditionally writes a four-word result to caller-provided storage. Depending on a normalized 32-bit selector, it:

- Copies four words from an object reached through the input object’s relationship field.
- Resolves another object through traversal or lookup, then copies that object’s four words.
- Delegates output handling to F-ca73cb512206.
- Returns without writing the output.

The supplied static evidence establishes selection, call ordering, pointer traversal, and output-copy behavior. It does not establish the semantic meaning of the four words, the object hierarchy, or the delegated output operation. Descriptions such as “relationship,” “lookup,” and “iterator” identify observed roles rather than verified application concepts.

## Interface

The function has four inputs and no explicit return value:

| Input | Established representation | Role |
|---|---|---|
| Input object | Address represented by a 32-bit machine word | Supplies an initial relationship pointer at byte offset `0x34`; also serves as an exact-match value during one traversal path. |
| Lookup/traversal argument | Signed 32-bit integer | Selects a lookup argument or controls relationship traversal, depending on the selector. |
| Selector | Unsigned 32-bit word, also interpreted as signed for dispatch | Determines whether to copy, resolve, delegate, or do nothing. |
| Output address | Pointer to writable 32-bit words | Receives four words on copy paths, or is passed to F-ca73cb512206 on delegated paths. |

The layout in the evidence uses four-byte words and pointers.

### Required layouts

The input object must permit a pointer read at byte offset `0x34`. This read occurs before selector handling, including for selectors that subsequently return without output.

An object selected for copying must permit four 32-bit reads:

| Byte offset | Output destination |
|---|---|
| `0x80` | Output offset `0x00` |
| `0x84` | Output offset `0x04` |
| `0x88` | Output offset `0x08` |
| `0x8C` | Output offset `0x0C` |

Objects followed during relationship traversal must permit a pointer read at byte offset `0x34`.

Objects used for indirect calls must have a readable dispatch-table pointer at object offset zero. The relevant dispatch-table slots are specified under Dependencies.

### Preconditions and ownership

There is no general null-pointer validation. The caller must provide readable object storage for every path actually taken and writable output storage whenever the selected behavior writes or delegates output handling.

No allocation or release operation is directly visible. The function passes existing pointers and addresses of temporary storage to dependencies. Ownership transfer, reference counting, retention, and release behavior of those dependencies are unknown.

## Behavior

### 1. Read the initial relationship pointer

Before examining the selector, the function reads the pointer at byte offset `0x34` of the input object. This becomes the initial selected object.

Consequently, an unsupported selector does not make an invalid input-object address safe.

### 2. Normalize and classify the selector

All selector constants and masks below are 32-bit values.

First, if the selector’s low eight bits are zero, the selector becomes zero. This rule takes precedence over all later rules, regardless of the original upper bits.

Next, the function tests whether:

- The selector AND `0xFFFFF0FF` equals `0x80000010`.

If so, the selector becomes exactly `0x80000010` and enters special dispatch. This test ignores bits 8 through 11, but not other differing bits.

Otherwise:

- A selector interpreted as signed and greater than zero goes directly to the four-word copy operation, using the initial selected object.
- A selector interpreted as signed and less than or equal to zero enters special dispatch.

Special dispatch clears bit `0x20` by applying the mask `0xFFFFFFDF`. The resulting selector is then handled as follows:

| Normalized special selector | Behavior |
|---|---|
| `0x00000000` | Resolve an object, then copy or delegate output handling. |
| `0x80000010` | Resolve an object using an alternate lookup path, then copy or delegate output handling. |
| `0x80000002` | Follow relationship pointers according to the second input, then copy. |
| `0x80000001` | Call F-ca73cb512206 with the output address. |
| `0x80000004` | Call F-ca73cb512206 with the output address. |
| `0x80000008` | Call F-ca73cb512206 with the output address. |
| Any other value | Return without writing the output. |

The signed comparison visible in the dispatch separates values around `0x80000008`; the table captures its effective outcomes.

Positive selectors bypass the bit-`0x20` clearing step. For example, positive `0x00000020` performs a direct copy rather than becoming selector zero.

### 3. Resolve an object for selectors zero and `0x80000010`

Both resolution modes first call F-583dc5fa4d66 with the initial selected object. This call happens even when the second input is nonzero and the returned pointer is not subsequently used by the visible path.

After that call, the function initializes a temporary result pointer to null.

#### Second input equals zero

The two resolution modes behave identically when the second input is zero:

1. Initialize temporary traversal state through F-538044b67754, passing:
   - The address of a 20-byte temporary state area.
   - The pointer returned by F-583dc5fa4d66.
   - The 32-bit value `0xFFFFFFFE`.
2. Repeatedly call F-6e4651b532dc with the state and a temporary candidate-output address.
3. If that call returns zero, stop searching without comparing the candidate.
4. If it returns nonzero, compare the candidate’s 32-bit value with the input-object address.
5. Continue until a candidate equals the input-object address or the traversal reports zero.
6. On a match, call F-ebd94610a438 with the traversal state and the temporary result-pointer address.
7. Call F-944b595e25e9 with the traversal state after either ordinary search termination.

A matching candidate ends the search even if F-ebd94610a438 produces a null result. There is no visible attempt to find another match.

After the traversal-state call to F-944b595e25e9:

- If the temporary result pointer is nonnull, select it for copying.
- Otherwise, invoke the indirect operation at dispatch-table byte offset `0x3C` on the object returned by F-583dc5fa4d66. Pass that object, `0x7FFFFFFF`, and the temporary result-pointer address.

The result pointer is examined after this fallback call. A null result leads to F-ca73cb512206; a nonnull result leads to copying.

The `0x3C` operation’s return value is not used. Its relevant visible result is the pointer written through the supplied output address.

#### Second input is nonzero

The operation is invoked on the initial selected object, not on the pointer returned by F-583dc5fa4d66.

- For selector zero, invoke the indirect operation at dispatch-table byte offset `0xE0`. Pass the selected object, an opaque value obtained from a fixed data location, and the second input.
- For selector `0x80000010`, invoke the indirect operation at dispatch-table byte offset `0xE4`. Pass the selected object and the second input.

In either case, treat the operation’s return value as the resolved object pointer:

- Null: call F-ca73cb512206 with the output address and return.
- Nonnull: copy the resolved object’s four words.

The identity and meaning of the fixed-data argument to the `0xE0` operation are not established.

### 4. Follow relationships for selector `0x80000002`

This path starts with the initial selected object.

If the second input is zero, no further relationship read occurs, and the function copies from that initial object.

For a positive second input of value \(n\), the function follows exactly \(n\) additional pointers. Each pointer is read at byte offset `0x34` of the currently selected object. It then copies from the resulting object.

The input object’s initial `0x34` read is separate from these \(n\) traversals. Thus, a second input of one selects the relationship of the initial selected object, rather than the initial selected object itself.

The loop decrements its signed 32-bit counter before testing against `-1`. Negative inputs do not receive a separate rejection or error path. Their interpretation across signed overflow is unresolved; they are not covered by an unconditional finite traversal-count guarantee.

There is no null check between relationship reads or before the final copy.

### 5. Copy four words

On every copy path, the selected object supplies four 32-bit words. The visible access order is:

1. Read selected-object offset `0x84`.
2. Read selected-object offset `0x88`.
3. Read selected-object offset `0x8C`.
4. Read selected-object offset `0x80` and write it to output offset `0x00`.
5. Write the previously read `0x84` value to output offset `0x04`.
6. Write the previously read `0x88` value to output offset `0x08`.
7. Write the previously read `0x8C` value to output offset `0x0C`.

All four source values are obtained before the first output write. For ordinary stable memory, overlapping source and destination storage therefore does not cause an earlier output write to replace a source value still awaiting a read.

This is not an atomic four-word update. Concurrency, volatile-memory semantics, and synchronization guarantees are not established.

## Side effects

### Direct effects

The function directly:

- Reads the input object’s relationship pointer unconditionally.
- Reads additional relationship pointers on the traversal-count path.
- Reads dispatch-table pointers and operation entries on indirect-call paths.
- Reads four object words and writes four output words on copy paths.
- Uses temporary state, candidate, and result-pointer storage during traversal.

Unsupported selectors perform no explicit output writes and invoke no dependencies after the initial relationship-pointer read.

### Delegated effects

F-ca73cb512206 receives the output address on explicit delegated selectors and on resolution failure. The supplied evidence does not show what it writes, whether it writes at all, or whether it affects other state. It must not be specified as “zeroing” or “clearing” the output without further evidence.

Other dependencies may mutate objects, globals, or indirectly reachable memory. The contract does not assume they preserve global state, retain no pointers, or release no memory.

The traversal-state finalization call is visible on ordinary traversal completion, including a successful match. No exception, trap, nonlocal-return, or dependency-failure cleanup behavior is established.

## Dependencies

| Dependency | Visible use | Unknowns |
|---|---|---|
| F-583dc5fa4d66 | Called with the initial selected object on both resolution selectors; returns a pointer used by the zero-second-input path. | Meaning, validity guarantees, ownership, side effects. |
| F-538044b67754 | Initializes traversal state using a returned object pointer and `0xFFFFFFFE`. | Traversal domain, ordering, resource acquisition, sentinel meaning. |
| F-6e4651b532dc | Advances traversal; zero stops, nonzero permits examination of the candidate output. | Candidate semantics, termination guarantees, other return-value meanings. |
| F-ebd94610a438 | Called on an exact candidate match; receives the temporary result-pointer address. | How it selects a result; whether null denotes failure or absence. |
| F-944b595e25e9 | Called after ordinary traversal termination. | Whether it releases resources or changes indirectly reachable state. |
| F-ca73cb512206 | Receives the caller’s output address on delegated-output paths. | Complete output and error semantics. |

Three additional operations have no supplied opaque function IDs:

- Dispatch-table slot `0x3C`: fallback pointer-output operation.
- Dispatch-table slot `0xE0`: nonzero-argument lookup with an additional opaque fixed-data argument.
- Dispatch-table slot `0xE4`: alternate nonzero-argument lookup.

These slots are invocation locations, not invented function identities. Their implementations and full contracts remain unknown.

## Acceptance examples

These examples are conditional static acceptance criteria, not executed tests. Except where stated otherwise, all referenced memory is valid, dependencies return normally, and memory remains stable between specified accesses.

1. **Positive selector copies directly.**  
   Given selector `0x00000001`, the initial selected object supplies the result. The second input does not affect selection. No resolution dependency is called.

2. **Positive bit `0x20` is not cleared.**  
   Given selector `0x00000020`, the function copies directly. It does not take selector-zero resolution.

3. **A zero low byte overrides upper bits.**  
   Given selector `0x80000100`, normalization produces zero. The function follows selector-zero resolution rather than returning as unsupported.

4. **Bits 8–11 may canonicalize alternate lookup.**  
   Given selector `0x80000F10`, the mask comparison produces `0x80000010`. With a nonzero second input, the function calls F-583dc5fa4d66 and then the `0xE4` operation on the initial selected object.

5. **Bit clearing is not followed by another canonicalization pass.**  
   Selector `0x80000130` does not satisfy the initial canonicalization test. Clearing `0x20` produces `0x80000110`, which is unsupported. The output remains unwritten by this function.

6. **Delegated output after bit clearing.**  
   Selector `0x80000021` becomes `0x80000001` during special dispatch. F-ca73cb512206 receives the output address; the four-word copy does not occur.

7. **Zero additional relationship steps.**  
   Selector `0x80000002` with second input zero copies from the pointer initially read at input-object offset `0x34`.

8. **Two additional relationship steps.**  
   The same selector with second input two performs two more `0x34` pointer reads before copying. No null-check fallback is promised.

9. **Traversal match with a nonnull result.**  
   For selector zero and second input zero, suppose traversal returns the input-object address and F-ebd94610a438 supplies a nonnull result. F-944b595e25e9 is called before that result is selected for copying. The fallback operation is skipped if the result remains nonnull.

10. **No match leads to fallback.**  
    If traversal returns zero before finding a match, F-944b595e25e9 is called, followed by the `0x3C` fallback because the temporary result pointer is null. A null fallback result causes F-ca73cb512206 to be called.

11. **Unsupported selector preserves output directly.**  
    Selector `0x80000003` returns without output writes or dependency calls, but still performs the initial input-object relationship read.

## Uncertainties

The following items are **documented_only**:

- Semantic names and units for the four copied words.
- The identities of the indirect-operation targets and the fixed-data argument.
- Dependency ownership, allocation, retention, release, and global-state behavior.
- Whether dependency failures are represented by null, other state, exceptions, or nonlocal exits beyond the visible checks.
- A complete termination and arithmetic contract for negative relationship-traversal counts. Under an additional 32-bit wrapping interpretation, a negative count would imply its unsigned 32-bit value in additional traversals; that interpretation is not established here.
- Thread safety, atomicity, reentrancy, and concurrent mutation behavior.
- Handling of invalid pointers, inaccessible memory, or malformed dispatch tables. No recovery is visible.

The apparent intent is to obtain four related-object attributes or delegate their unavailable-result handling. That interpretation is inferred, not a substitute for the established word-level behavior.

## Evidence

This contract is derived solely from the supplied static function body and supplied opaque dependency mapping.

Established evidence includes the selector masks and constants, signed dispatch comparisons, relationship and result offsets, dependency-call ordering, indirect-call slots, null-result branches, and four-word read/write ordering.

No dependency implementations were supplied. No runtime tests have been performed, and none of the acceptance examples is claimed as an observed execution.
