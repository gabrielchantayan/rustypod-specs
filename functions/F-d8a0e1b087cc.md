<!-- Trusted provenance: F-d8a0e1b087cc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d8a0e1b087cc

## Purpose

F-d8a0e1b087cc conditionally dispatches an indirect callback to objects in up to thirteen indexed collections. A 32-bit mask in a caller-supplied context selects the collections. Selected collections are visited in ascending index order, and their entries are visited in ascending address order.

For each visited entry, the function obtains an object address and dispatches through the first function-pointer slot of a table referenced by that object. The callback receives the object address and the original context address.

The established behavior is ordered dispatch, not filtering or transformation of the collection contents. An interpretation as event delivery to subscribers is plausible but **inferred intent**, not part of the established contract.

## Interface

### Inputs

The function accepts two address-bearing arguments represented as 32-bit integers in the supplied evidence:

1. **Collection descriptor base.** The address of a sequence of thirteen descriptors, each occupying 12 bytes.
2. **Context address.** An address passed unchanged to every invoked callback. The function also reads a 32-bit selection mask at byte offset 16 from this address.

These representations support a 32-bit address model. Behavior on a different pointer-width ABI is not established.

### Descriptor layout

For collection index \(i\), where \(0 \leq i \leq 12\), the descriptor starts at:

\[
\text{collection descriptor base} + 12i
\]

| Byte offset within descriptor | Established use |
|---|---|
| 0–3 | Starting address of a contiguous sequence of 4-byte entries |
| 4–7 | Exclusive ending address of that sequence |
| 8–11 | Not directly accessed by this function |

Each entry is a 4-byte object address. The ending address denotes a boundary, not an entry to invoke.

The third descriptor word may have a role elsewhere, but this function does not establish one. In particular, treating it as a capacity or allocation boundary would be an unsupported interpretation.

### Object and dispatch layout

For each visited entry:

- The entry supplies an object address.
- The first 4-byte word at that object address supplies a dispatch-table address.
- The first 4-byte word at that table address supplies the callback target.
- The callback is invoked with the object address as its first argument and the original context address as its second argument.

This resembles a first-slot virtual dispatch, but no object model, class identity, or callback meaning is established.

### Output

The function has no explicit return value. Callback return values, if any, are not used.

Normal completion means all selected collections have reached their observed ending addresses and all invoked callbacks have returned normally. No status indicating invocation count, skipped collections, or failure is produced.

### Preconditions

For defined, normally completing behavior:

- The context mask must be readable as a 32-bit word whenever the function reads it.
- Each selected descriptor must provide readable start and end words.
- Every entry actually visited must be readable as a 4-byte object address.
- Each object, dispatch table, and callback target used for an invocation must be valid for that operation.
- Callback targets must accept the two supplied arguments under the applicable ABI.
- For termination, each traversal must eventually encounter an observed end address exactly equal to its current cursor.
- Callback-induced changes must preserve the validity of all subsequent reads and calls.

No explicit validation enforces these conditions.

## Behavior

### Selection and collection order

The function considers exactly thirteen collection indices: 0 through 12, in increasing order.

At each index, it reads the 32-bit mask at context offset 16 and tests the bit corresponding to that index. It does not take one mask snapshot for the entire operation.

Consequences:

- Bits 0 through 12 select the corresponding collections.
- Bits 13 through 31 do not select any collection in this invocation.
- A clear bit causes the corresponding descriptor and entries to be skipped.
- A set bit causes the corresponding descriptor to be traversed.
- A callback may affect whether a later collection is selected by changing the mask before that later index is examined.
- Changing the mask does not cancel traversal of a collection already selected.
- Changes affecting previously examined indices do not cause those indices to be revisited.

The shift used to select a bit is limited to positions 0 through 12. No shift by the word width, negative shift, or sign-bit shift is needed in this function.

### Beginning a selected collection

After selecting a collection, the function computes its descriptor address using the 12-byte stride and reads its starting entry address.

That starting address initializes a traversal cursor once for the collection. A callback that later changes the descriptor’s start word does not cause this cursor to restart or move to the new start.

The function compares the cursor with the descriptor’s end word before visiting an entry. If they are equal, the collection is empty for this traversal and no callback is invoked.

The function does not check whether the end address is numerically greater than the start address.

### Visiting entries

When the cursor and observed end address differ, the function dispatches a callback for the entry at the cursor:

- It obtains the object through the current 4-byte entry.
- It resolves the target through the object’s first word and the referenced table’s first word.
- It invokes the target with the object and context addresses.
- After the callback returns, it advances the cursor by 4 bytes.
- It checks the descriptor’s end word again before visiting another entry.

For a stable, valid range of \(n\) entries, the object slots are visited at offsets \(0, 4, 8, \ldots, 4(n-1)\) from the starting address.

There is no deduplication. Repeated object addresses result in repeated invocations. The same object appearing in different selected collections is also invoked once for each visited occurrence.

### End-boundary changes during callbacks

The descriptor’s end word is consulted on every loop condition, including after callbacks. It is not a fixed traversal limit captured at collection entry.

Thus, conditional on valid memory and stable changes:

- Moving the end farther forward can make additional entries eligible for traversal.
- Moving the end to the next cursor position can stop traversal after the current callback.
- Moving the end behind the cursor does not itself terminate traversal.
- Replacing the underlying storage does not automatically relocate the cursor.

The stopping condition is address equality, not a less-than comparison. A malformed or dynamically altered range may therefore continue into unrelated memory, fault, or fail to terminate. There is no bounds-recovery behavior.

### Integer widths and address calculations

The relevant quantities in the supplied representation are 32-bit words and addresses. Descriptor indexing adds \(12i\), with a maximum descriptor-start displacement of 144 bytes. Traversal advances by 4 bytes per visited entry.

A valid stable range has an end reachable from its start by successive 4-byte advances. Ordinary safe use requires the traversed storage to remain within a valid allocation without address wraparound.

The supplied representation does not establish a portable overflow policy for malformed addresses. Address wrapping must not be treated as a supported way to reach an otherwise inconsistent end boundary.

Byte order, alignment tolerance, and exact calling-convention details are not established by this function alone.

### Completion and failure

After a selected collection finishes, the function proceeds to the next index. After index 12 has been processed or skipped, it returns.

There is no explicit branch for a null object, invalid table, missing callback, inaccessible mask, malformed range, or callback failure. There is also no retry, rollback, failure count, or local cleanup mechanism.

A callback that does not return prevents ordinary continuation. The behavior of faults, exceptions, and nonlocal control transfers depends on the surrounding runtime and is not specified here.

## Side effects

### Direct effects

The function performs reads and indirect calls. The supplied evidence contains no direct writes to:

- The mask or other context fields.
- Descriptor words.
- Entry storage.
- Object or dispatch-table storage.
- Global storage.

Its direct traversal state consists of the collection index and current cursor. No direct allocation or deallocation is present.

### Callback effects

Callbacks are unrestricted by the available evidence. They may alter the context, descriptors, entries, object state, global state, or allocation state.

No assumption is made that a callback:

- Preserves the selection mask.
- Preserves descriptor boundaries.
- Leaves the current or subsequent objects alive.
- Preserves globals.
- Retains or releases ownership of any argument.
- Returns normally.

Any callback behavior that invalidates a later read or dispatch can prevent normal completion.

### Observable ordering

The established control-flow order is:

1. Examine collection indices in ascending order.
2. Read and test the mask for the current index.
3. For a selected collection, obtain its start and compare against its end.
4. Invoke the callback for the current entry.
5. Wait for that invocation to return before advancing to another entry.
6. Advance the cursor and consult the end again.
7. Finish the collection before examining the next collection’s mask bit.

The dispatch expression uses the current entry both to resolve the callback target and to supply the object argument. The exact number and relative ordering of the underlying entry loads are not established as an externally reliable contract. Concurrent mutation, volatile-memory semantics, and synchronization guarantees are **documented_only** unknowns.

### Ownership

The function itself neither acquires nor releases storage in the supplied evidence. Passing an address to a callback does not, by itself, establish ownership transfer.

Whether callbacks may retain addresses or release referenced storage is unknown. Such actions are compatible with this traversal only when all subsequent accesses remain valid.

## Dependencies

### F-indirect-01

**Document-local opaque identifier:** the dynamically resolved callback target. This is not an externally established identity, and different entries may resolve to different functions.

- **Resolution:** first function-pointer slot of the table referenced by the object’s first word.
- **Arguments:** current object address, then original context address.
- **Invocation:** once per visited entry whose dispatch completes.
- **Return value:** ignored.
- **State preservation:** unknown.
- **Memory ownership:** unknown.
- **Failure behavior:** unknown.

No named direct function dependency is established. The callback implementations are not available in the supplied evidence.

## Acceptance examples

These examples are conditional static acceptance criteria, not executed tests. Except where mutation is stated explicitly, they assume valid 32-bit addresses, stable descriptors and entries, and normally returning callbacks.

### All selection bits clear

If the mask is zero throughout the invocation, no callbacks occur. The function still examines all thirteen collection indices. It does not directly read their descriptor contents.

### Only an unsupported high bit set

If the mask contains only bit 20, no collection is traversed. That bit is outside the thirteen supported selection positions.

### One selected, empty collection

If bit 4 is set and collection 4 has equal start and end addresses, collection 4 produces no callback. No entry at that address is dereferenced for that collection.

### Ordered dispatch across collections

If only collections 1 and 3 are selected, collection 1 contains objects A and B, and collection 3 contains object C, the invocation order is A, B, C. Every call receives the same original context address as its second argument.

### Duplicate object entries

If a selected collection contains the same valid object address twice, that object’s resolved callback is invoked twice, once for each entry. The function does not collapse duplicates.

### Mask changed by an earlier callback

If collection 0 is selected and one of its callbacks sets bit 5 before index 5 is examined, collection 5 is selected when reached, provided the bit remains set at that read. Clearing bit 0 during the callback does not stop the already active traversal of collection 0.

### Start changed during traversal

If a callback changes its current descriptor’s start word while preserving the current storage and end boundary, traversal continues from the existing cursor plus 4 bytes. It does not restart at the replacement start.

### End extended during traversal

If a callback extends the current end from immediately after the current entry to immediately after one additional valid entry, that additional entry is visited, provided the updated end remains in effect.

### End moved behind the cursor

If a callback moves the end behind the next cursor position, successful termination is not guaranteed. The function does not interpret “past the end” as completion; it requires exact equality with an observed end address.

## Uncertainties

The following are **documented_only** unknowns:

- The semantic meanings of collections, objects, and callbacks.
- The purpose of each descriptor’s third word.
- The context layout outside the mask at offset 16.
- Callback identities, return types, ownership rules, and side effects.
- Whether the surrounding system permits descriptor mutation during dispatch.
- Whether synchronization outside this function prevents concurrent changes.
- Exact machine-load behavior within the indirect-call expression.
- Endianness, alignment requirements, and precise ABI conventions.
- Runtime handling of invalid accesses and nonlocal callback exits.

These unknowns do not weaken the established ordered traversal behavior under valid, stable inputs. They limit claims about mutation safety, portability, ownership, and failure recovery.

## Evidence

This contract is based solely on static examination of the supplied function representation.

Established evidence supports thirteen ascending selection checks, a 32-bit mask at context offset 16, 12-byte descriptor spacing, 4-byte entry advancement, repeated end-boundary checks, first-slot indirect dispatch, and forwarding of the object and context addresses.

The evidence also supports the absence of direct stores, direct allocation or release calls, explicit validation branches, and callback-result handling within this function.

Subscriber dispatch and virtual-method terminology are interpretations, not required semantics. No implementation of an indirect target was supplied. No runtime tests have been performed.
