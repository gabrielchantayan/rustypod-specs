<!-- Trusted provenance: F-ee9f9ccf719b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ee9f9ccf719b

## Purpose

F-ee9f9ccf719b initializes three 32-bit values in local storage, exposes that storage to a preparation dependency, selects an inline-object update, and issues eight ordered indirect calls.

The update is selected by reading the seventh 32-bit integer in the object, at byte offset 24, converting its value to `char`, and comparing the converted value with zero. This is not independently established as a byte read at offset 24.

On normal completion, the function returns a 64-bit bit pattern containing the final first staged word in bits 63–32 and zero in bits 31–0. Results from the preparation dependency, conditional external dependency, and indirect calls are not used.

An apparent purpose is to prepare a compact representation, place some of it in an object, and issue a fixed operation sequence. The representation’s meaning, the operations’ meanings, and the conditional external dependency’s copying semantics remain unestablished.

## Interface

### Inputs

The function receives four inputs:

| Input | Established representation and use |
|---|---|
| Object reference | Address represented using a 32-bit address model; used for integer access, inline storage, and indirect dispatch |
| First value | Unsigned 32-bit value |
| Second value | 32-bit value with unspecified semantic type |
| Third value | 32-bit value with unspecified semantic type |

The relevant object layout is:

| Byte offset | Source-visible use |
|---|---|
| 0–3 | Dispatch-table address |
| 24–27 | Seventh 32-bit integer; its value is converted to `char` for branch selection |
| 26 | Destination of the direct one-byte store |
| 26 onward | Destination passed to the conditional external dependency |

The mode-selection integer and destination overlap. In particular, offset 26 lies within the integer at offsets 24–27. A conventional ten-byte copy to offsets 26–35 would also overwrite its final two address-ordered bytes.

The object’s complete layout and size are unknown. These accesses do not establish a distinct one-byte mode field at offset 24.

### Staged storage

Before preparation, three initialized 32-bit words occupy a contiguous 12-byte local region:

| Relative byte offsets | Initial contents |
|---|---|
| 0–3 | First input value |
| 4–7 | Second input value |
| 8–11 | Third input value |

The words use the target’s byte order, which is not independently established here.

F-3b3feb156f36 receives the address of the first staged word without an explicit length. Twelve initialized bytes are visible, but that does not establish the dependency’s complete access requirements or bounds.

### Return value

On normal completion, the return bit pattern is the final first staged unsigned 32-bit word widened to unsigned 64 bits and shifted left by 32:

- Bits 63–32 contain all 32 bits of that word.
- Bits 31–0 are zero.

The displayed return type is signed 64-bit, although the packing operation uses an unsigned 64-bit value. A result whose top bit is set is negative under the usual signed 64-bit interpretation. Exact language-level conversion rules are not independently established.

The returned word is read after the eighth indirect call. It need not equal either the original first input or the value used during the earlier inline update.

### Preconditions

Normal completion requires:

- Valid object access for the source-level 32-bit integer read at offsets 24–27, with any alignment requirements imposed by the target.
- A writable byte at offset 26 on the direct-store branch.
- A destination satisfying F-000000000001’s requirements on the external-call branch. If that dependency performs a conventional ten-byte copy, offsets 26–35 must be writable.
- Readable current dispatch state before each indirect call: the object’s dispatch-table address and the callable entry at table byte offset 88.
- Callable targets compatible with the displayed three-argument invocation.
- Compliance with each dependency’s otherwise unknown memory, lifetime, and execution requirements.

Requiring only a readable byte at offset 24 is insufficient to describe the displayed integer access. The source-level access does not, by itself, establish the exact machine instruction or physical load width.

No null checks, bounds checks, dispatch validation, or explicit error handling are visible.

## Behavior

### Preparation precedes branch selection

The function initializes all three staged words and calls F-3b3feb156f36 with the first word’s address.

Only after that call returns does it access the integer used for branch selection. Preparation can therefore affect the staged values and, if it has access to the object through other reachable state, the integer subsequently examined.

The function does not restore the staged values after preparation. The preparation dependency’s result is unused.

### Branch selection uses integer-to-character conversion

The function accesses the seventh 32-bit integer in the object, occupying offsets 24–27. It converts that integer value to `char`, then compares the converted value with zero.

The branch depends on the converted value, not directly on whether:

- The full integer is zero.
- The first address-ordered byte at offset 24 is zero.
- Any particular memory byte is zero independently of target conversion and byte-order semantics.

Under conventional eight-bit truncating conversion, the converted value is zero exactly when the integer’s low eight bits are zero. Under those assumptions, every nonzero converted value selects the other branch, regardless of character signedness.

Mapping this conversion to an address-ordered byte additionally requires known byte order. With conventional eight-bit truncation:

- On a little-endian target, the low eight bits reside at offset 24.
- On a big-endian target, they reside at offset 27.

These are conditional interpretations, not an established target identification.

### Zero converted value: invoke the external dependency

When the converted character compares equal to zero, the function invokes F-000000000001 with:

1. Destination address: object address plus 26 bytes.
2. Source address: beginning of the staged region.
3. Count-like argument: decimal 10.

The dependency’s result is unused.

The established behavior is this invocation. A conventional copying interpretation remains conditional.

If the dependency copies ten consecutive bytes, staged offsets 0–9 are transferred to object offsets 26–35. This comprises both complete first and second staged words and the first two address-ordered bytes of the third word. It excludes staged offsets 10–11.

Such a copy would overwrite object offsets 26–27 within the integer already used for selection. The branch is not reconsidered afterward.

### Nonzero converted value: store one byte directly

When the converted character does not compare equal to zero, the function directly stores the low eight bits of the current first staged word at object offset 26.

This store is a one-byte numeric truncation. It is not generally equivalent to copying the first address-ordered byte of the staged word; that equivalence requires suitable byte-order semantics.

No other direct inline-destination store is visible in this branch. The store overlaps the previously read integer, but the function does not read that integer again for branch selection.

Dependencies and subsequent callbacks may change the destination independently.

### Retain one global argument value

After the selected update completes, the function reads a 32-bit global slot and retains its value for later calls.

This contract labels the ten distinct source-visible global argument slots **G1** through **G10**. The labels imply neither application meaning nor preservation guarantees.

The retained value comes from G1. Calls 3–8 use that retained value even if later execution changes G1.

### Perform eight ordered indirect calls

Each indirect call receives the object reference and two 32-bit arguments:

| Call | Second argument | Third argument |
|---|---|---|
| 1 | Separate read of G1 | G2 |
| 2 | G3 | G4 |
| 3 | Retained G1 value | G5 |
| 4 | Retained G1 value | G6 |
| 5 | Retained G1 value | G7 |
| 6 | Retained G1 value | G8 |
| 7 | Retained G1 value | G9 |
| 8 | Retained G1 value | G10 |

The retained G1 read precedes call 1’s argument reads. Call 1 references G1 separately rather than explicitly using the retained value.

The calls execute sequentially. Later argument globals are consumed after earlier calls return; they are not all snapshotted before the sequence. An earlier callback can therefore change values used by a later call.

For each call, the target is selected through the object’s then-current dispatch-table address and the entry at table byte offset 88. Changes to either location can redirect subsequent calls.

The supplied expression does not establish relative evaluation order among the target lookup and individual argument reads within a single call. No stronger ordering among those reads is required by this contract.

All callback results are ignored. There is no visible result-dependent retry, early exit, or rollback.

### Construct the return value

After call 8 returns, the function reads the first staged unsigned 32-bit word, widens it to unsigned 64 bits, and shifts it left by 32.

The shift operates at 64-bit width. It does not shift a 32-bit operand by its width or discard the word being returned.

The function makes no direct assignment to that staged word after initialization. Nevertheless, preparation receives its address, the conditional external dependency receives the same address, and legitimately retained access could permit later changes during the invocation. No dependency contract establishes preservation.

## Side effects

### Directly established effects

The function:

- Initializes 12 bytes of local staged storage.
- Exposes the first staged word’s address to preparation.
- Accesses the 32-bit integer at object offsets 24–27 after preparation and converts its value to `char` for the branch.
- Either stores one byte at object offset 26 or invokes the conditional external dependency.
- Reads the global argument slots, including separate retained and call-1 uses of G1.
- Performs eight sequential indirect calls.
- Reads the first staged word to construct the return value.

No direct writes to the global argument slots are visible.

### Dependency-mediated effects

The evidence does not establish that any dependency preserves:

- The staged words.
- The object’s selection integer, inline destination, or dispatch state.
- The global argument slots.
- Other reachable state.

Nor does it establish that dependencies avoid allocation, release, I/O, reentrancy, or abnormal termination.

Consequently, the function’s direct store or conditional invocation does not determine the destination’s final contents. Eight callbacks execute afterward on a normally completing path.

### Lifetime and ownership

The staged region is local to the invocation. Passing its address does not visibly transfer ownership or extend its lifetime beyond that invocation.

No explicit object allocation, retain, release, or ownership transfer appears in the function. Dependency-mediated ownership changes remain documented_only. Continued object validity cannot be inferred merely because a dependency returns.

## Dependencies

### F-3b3feb156f36

The supplied identity corresponds to the preparation dependency.

- Receives the first staged word’s address.
- Executes before branch-selection integer access.
- Has no result consumed by this function.
- Has unknown access bounds, mutations, ownership effects, and failure behavior.

### F-000000000001

This locally assigned opaque ID denotes the external dependency called on the zero-conversion branch. It is not a supplied provenance identity.

- Receives the inline destination, staged-region address, and 10.
- Has no result consumed by this function.
- Has a call shape consistent with copying, but copying is not established.
- Has unknown overlap, mutation, ownership, and failure semantics.

### F-000000000002

This locally assigned opaque ID denotes the dynamic callable role at dispatch-table byte offset 88, not one necessarily stable function.

- Is invoked eight times on a normally completing path.
- Receives the arguments listed above.
- Has no result consumed by this function.
- Can resolve to different concrete targets as dispatch state changes.

## Acceptance examples

These are conditional static acceptance examples, not executed tests.

1. **Nonzero character conversion selects the direct store.**  
   Assume conventional eight-bit truncating character conversion. If the selection integer is hexadecimal `00000001` after preparation and the first staged word is `123456AB`, the function stores `AB` at object offset 26 and does not invoke F-000000000001. This holds for either byte order under the stated conversion assumption.

2. **A nonzero selection integer can select the zero branch.**  
   Under the same conversion assumption, integer value `00000100` converts to zero. The function invokes F-000000000001 rather than performing the direct store.

3. **A nonzero byte at offset 24 need not select the direct store.**  
   On a big-endian target with conventional eight-bit truncation, integer value `01000000` has byte `01` at offset 24 but converts to zero. The external-call branch applies. Treating offset 24 as an independently tested byte would produce the wrong result.

4. **Preparation determines the value tested.**  
   Assume conventional eight-bit truncation. If preparation changes the selection integer from `00000001` to `00000100`, the external-call branch applies. Deciding the branch before preparation would violate the visible ordering.

5. **The external invocation has a ten-byte count-like argument.**  
   If the converted selection value is zero, F-000000000001 receives object offset 26, the staged address, and 10. If independently established to copy bytes, it transfers exactly staged offsets 0–9, not all twelve initialized bytes.

6. **An overlapping write does not trigger reselection.**  
   If the direct store at offset 26 changes the selection integer’s later value, the function still proceeds to global reads and callbacks. It does not test the integer again.

7. **Retained and separate G1 reads can differ conditionally.**  
   If a defined mechanism changes G1 from A to B between retention and call 1’s separate read, call 1 receives B, while calls 3–8 receive A. This example establishes no concurrency or synchronization guarantee.

8. **Earlier callbacks affect later reads.**  
   If call 1 changes G3 and G4 and returns normally, call 2 uses their later values. If it replaces the dispatch table with another valid table, call 2 selects its target from that replacement.

9. **Return packing preserves the final word.**  
   A final first staged word of `89ABCDEF` produces return bits `89ABCDEF00000000`. A final zero word produces zero. Neither value causes the eight-call sequence to be skipped.

## Uncertainties

The following remain documented_only:

- Application meanings of the object, selection integer, inline representation, globals, and callbacks.
- Target byte order, exact character-conversion rules, character signedness, and ABI details beyond displayed widths and offsets.
- Exact machine access width implementing the source-level integer access.
- Complete object size and alignment.
- Preparation’s access bounds and effects.
- Whether F-000000000001 copies memory, and its overlap or fault behavior.
- Concrete callback identities and their contracts.
- Dependency-mediated ownership or lifetime changes.
- Permitted concurrency and synchronization.
- Exceptional control flow and recovery behavior.

No status-to-error mapping is visible. Ordinary dependency return values do not cause explicit local error handling because they are unused. A dependency that fails to return normally can prevent later operations and the final return; no local recovery is shown.

## Evidence

Established static evidence consists of the supplied function body and the supplied identity mapping for F-3b3feb156f36.

That evidence supports the staged widths and layout, preparation ordering, seventh-integer access followed by character conversion, overlapping destination offsets, conditional invocation or byte store, global argument sequence, repeated dispatch selection, unused dependency results, and 64-bit return packing.

A byte test specifically at object offset 24 is not independently supported. Equivalence to such a test requires additional target semantics. Copying, representation meaning, and dependency guarantees remain conditional or unknown.

No runtime tests have been performed.
